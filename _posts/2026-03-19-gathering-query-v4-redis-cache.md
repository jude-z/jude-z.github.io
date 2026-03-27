---
title: "모임 조회 4단계 — Redis 캐시 + Redisson 분산 락으로 p50 11ms 달성"
project: "gathering-server"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Redis
  - Redisson
  - Locust
date: 2026-03-19
excerpt: "카테고리 수만큼 쿼리가 발생하는 구조적 한계를 Redis 캐시 + Redisson 분산 락 + DB fallback으로 해결한 과정."
---

> **모임 조회 API 시리즈**
> [1단계. 튜닝 없이 v1 쿼리 분석 — 389초](/projects/gathering-query-v1-no-tuning/)
> [2단계. 쿼리 분리 + 복합 인덱스 — 389초 → 5초](/projects/gathering-query-v2-separation/)
> [3단계. ROW_NUMBER 제거 — 5초 → 317ms](/projects/gathering-query-v3-remove-rownum/)
> 4단계. Redis 캐시 + Redisson 분산 락 — p50 11ms *(현재 글)*

---

## Problem

3단계에서 복합 인덱스로 개별 쿼리는 0.17ms까지 줄였지만, **카테고리 수만큼 쿼리가 발생하는 구조적 한계**가 남아 있었다.

Locust 100명 부하테스트:
- 28 RPS, 0% 실패
- **p50 1,600ms / p95 1,900ms**

카테고리가 20개면 쿼리 20번 + enrollment 배치 1번 = 21번의 DB 왕복이 매 요청마다 발생한다. 개별 쿼리가 아무리 빨라도, 동시 100명이 각각 21번씩 DB를 때리면 커넥션 풀과 네트워크 오버헤드가 병목이 된다.

모임 메인 페이지는 **모든 사용자에게 동일한 결과**를 보여준다. 사용자별 개인화가 없으므로, 캐싱에 이상적인 엔드포인트다.

## Action

### 캐시 설계: Redis + DB fallback + Redisson 분산 락

단순히 Redis에 캐싱하면 끝나는 게 아니다. 세 가지 문제를 동시에 해결해야 한다:

1. **캐시 스탬피드** — TTL 만료 시 동시 100명이 한꺼번에 DB를 조회
2. **Redis 장애** — Redis가 내려가면 캐시 자체가 불가능
3. **데이터 정합성** — 캐시가 너무 오래되면 사용자 경험 저하

### 1. Soft TTL + Hard TTL: stale-while-revalidate

```java
private static final Duration CACHE_TTL = Duration.ofMinutes(60);   // Hard TTL
private static final Duration SOFT_TTL = Duration.ofMinutes(55);    // Soft TTL
```

- **Hard TTL 60분**: Redis 키 자체의 만료 시간
- **Soft TTL 55분**: 남은 TTL이 5분 이하일 때 **한 스레드만 선점하여 갱신**

현재 요청은 기존 캐시(약간 stale)를 그대로 반환하고, 백그라운드에서 갱신이 완료되면 다음 요청부터 새 데이터를 받는다.

```java
private void triggerRefreshIfNeeded(Supplier<List<MainGatheringsProjection>> dbLoader) {
    try {
        Long remainTtl = redisTemplate.getExpire(CACHE_KEY, TimeUnit.MINUTES);
        if (remainTtl == null || remainTtl > (CACHE_TTL.toMinutes() - SOFT_TTL.toMinutes())) {
            return;  // 아직 여유 있음
        }

        // 남은 TTL 5분 이하 → 하나의 스레드만 갱신
        RLock lock = redissonClient.getLock(LOCK_KEY);
        boolean acquired = lock.tryLock(0, LOCK_LEASE_TIME, TimeUnit.SECONDS);
        if (acquired) {
            try {
                List<MainGatheringsProjection> result = dbLoader.get();
                String json = serialize(result);
                redisTemplate.opsForValue().set(CACHE_KEY, json, CACHE_TTL);
                saveToDbCache(json);
            } finally {
                lock.unlock();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } catch (Exception e) {
        log.warn("Cache refresh failed", e);
    }
}
```

`tryLock(0, ...)` — wait 시간 0초. 락을 잡지 못하면 즉시 포기한다. 이미 다른 스레드가 갱신 중이므로, 나머지 스레드는 기존 캐시를 반환하면 된다.

### 2. Redisson 분산 락: 캐시 미스 시 보호

캐시 미스(Redis에 키가 없는 상태)일 때, 동시 100명이 한꺼번에 DB를 조회하는 **캐시 스탬피드**를 방지한다.

```java
private List<MainGatheringsProjection> loadWithLock(Supplier<List<MainGatheringsProjection>> dbLoader) {
    RLock lock = redissonClient.getLock(LOCK_KEY);
    try {
        boolean acquired = lock.tryLock(3, LOCK_LEASE_TIME, TimeUnit.SECONDS);
        if (acquired) {
            try {
                // Double-check: 다른 스레드가 이미 채웠을 수 있음
                String cached = redisTemplate.opsForValue().get(CACHE_KEY);
                if (cached != null) return deserialize(cached);

                List<MainGatheringsProjection> result = dbLoader.get();
                String json = serialize(result);
                redisTemplate.opsForValue().set(CACHE_KEY, json, CACHE_TTL);
                saveToDbCache(json);
                return result;
            } finally {
                lock.unlock();
            }
        } else {
            // 락 획득 실패 → 다른 스레드가 채운 캐시 확인
            String cached = redisTemplate.opsForValue().get(CACHE_KEY);
            if (cached != null) return deserialize(cached);
            return getFromDbCacheOrLoad(dbLoader);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return getFromDbCacheOrLoad(dbLoader);
    } catch (Exception e) {
        log.warn("Redis failed during loadWithLock, fallback to DB cache", e);
        return getFromDbCacheOrLoad(dbLoader);
    }
}
```

- `tryLock(3, 30, SECONDS)` — 최대 3초 대기, 30초 lease
- **Double-check locking**: 락 획득 후 Redis를 다시 확인. 대기 중에 다른 스레드가 캐시를 채웠을 수 있다.

### 3. DB 캐시 fallback: Redis 장애 대비

Redis가 완전히 내려간 상황에서도 서비스가 동작해야 한다. `gathering_cache` 테이블에 동일한 데이터를 저장한다.

```java
private void saveToDbCache(String json) {
    try {
        String sql = "INSERT INTO gathering_cache (cache_key, data, expired_at, created_at) " +
                "VALUES (?, ?, ?, ?) " +
                "ON DUPLICATE KEY UPDATE data = VALUES(data), " +
                "expired_at = VALUES(expired_at), created_at = VALUES(created_at)";
        jdbcTemplate.update(sql, CACHE_KEY, json,
                LocalDateTime.now().plus(CACHE_TTL), LocalDateTime.now());
    } catch (Exception e) {
        log.warn("DB cache save failed", e);
    }
}

private List<MainGatheringsProjection> getFromDbCacheOrLoad(
        Supplier<List<MainGatheringsProjection>> dbLoader) {
    try {
        String sql = "SELECT data FROM gathering_cache " +
                "WHERE cache_key = ? AND expired_at > NOW() LIMIT 1";
        List<String> results = jdbcTemplate.queryForList(sql, String.class, CACHE_KEY);
        if (!results.isEmpty()) {
            return deserialize(results.get(0));
        }
    } catch (Exception e) {
        log.warn("DB cache read failed", e);
    }
    return dbLoader.get();  // 최종 fallback: 직접 DB 조회
}
```

`ON DUPLICATE KEY UPDATE`로 캐시 키 기준 upsert. Redis 장애 시 이 테이블에서 만료되지 않은 데이터를 반환한다. DB 캐시마저 없으면 직접 DB 조회로 최종 fallback한다.

### 4. 메인 진입점: getOrLoad

```java
public List<MainGatheringsProjection> getOrLoad(Supplier<List<MainGatheringsProjection>> dbLoader) {
    // 1. Redis 시도
    try {
        String cached = redisTemplate.opsForValue().get(CACHE_KEY);
        if (cached != null) {
            cacheHitCounter.increment();
            triggerRefreshIfNeeded(dbLoader);
            return deserialize(cached);
        }
    } catch (Exception e) {
        cacheMissCounter.increment();
        log.warn("Redis read failed, fallback to DB cache", e);
        return getFromDbCacheOrLoad(dbLoader);
    }

    // 2. 캐시 미스 - 락 잡고 로드
    cacheMissCounter.increment();
    return loadWithLock(dbLoader);
}
```

### 5. Micrometer 캐시 히트율 모니터링

```java
this.cacheHitCounter = Counter.builder("gathering_cache_hit")
        .description("Gathering cache hit count")
        .register(meterRegistry);
this.cacheMissCounter = Counter.builder("gathering_cache_miss")
        .description("Gathering cache miss count")
        .register(meterRegistry);
```

Prometheus/Grafana에서 캐시 히트율을 실시간으로 모니터링할 수 있다.

### GatheringService: v4

```java
public ApiResponse gatheringsV4() {
    List<MainGatheringsProjection> mainGatheringElements =
        gatheringCacheService.getOrLoad(this::loadGatheringsFromDb);
    Map<String, CategoryTotalGatherings> map = categorizeByCategory(mainGatheringElements);
    return toMainGatheringResponse(map);
}
```

`loadGatheringsFromDb()`는 v3의 로직과 동일하다. 캐시 hit 시에는 DB 쿼리가 **0번** 발생한다.

## Result

### Locust 100명 부하테스트

![v4 부하테스트](/assets/images/gathering/네번쨰%20부하테스트.png)

- **49.4 RPS, 0% 실패**
- **p50 11ms / p95 15ms**

### 전체 시리즈 비교

| 지표 | v1 (튜닝 없음) | v2 (쿼리 분리) | v3 (ROW_NUMBER 제거) | v4 (캐시) |
|---|---|---|---|---|
| EXPLAIN ANALYZE | 389초 | 5초 | 0.17ms | 캐시 hit |
| Locust RPS | - | 0.5 | 28 | **49.4** |
| 실패율 | - | 61% | 0% | **0%** |
| p50 | 7분 38초 | 84초 | 1,600ms | **11ms** |
| p95 | - | 84초 | 1,900ms | **15ms** |
| HikariCP Pending | - | 91 | 33 | **0** |
| DBLoadCPU | - | 2.47 | 0.12 | **~0** |

## Reflection

쿼리 튜닝만으로는 한계가 있다. 복합 인덱스로 개별 쿼리를 0.17ms까지 줄여도, **카테고리 수만큼 쿼리가 발생하는 구조적 문제**는 남는다. 이 한계를 인식하고 캐시 레이어를 도입한 것이 핵심 판단이었다.

캐시를 도입할 때 가장 중요한 질문은 "Redis가 죽으면?"이다. Redisson 분산 락으로 스탬피드를 막고, DB 캐시 테이블로 Redis 장애에 대비하고, Micrometer로 히트율을 모니터링하는 **3중 안전장치**가 운영 환경에서의 신뢰성을 보장한다.

---

> **모임 조회 API 시리즈 완결**
> [1단계. 튜닝 없이 v1 쿼리 분석 — 389초](/projects/gathering-query-v1-no-tuning/)
> [2단계. 쿼리 분리 + 복합 인덱스 — 389초 → 5초](/projects/gathering-query-v2-separation/)
> [3단계. ROW_NUMBER 제거 — 5초 → 317ms](/projects/gathering-query-v3-remove-rownum/)
> 4단계. Redis 캐시 + Redisson 분산 락 — p50 11ms *(현재 글)*
