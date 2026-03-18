---
title: "Redis Cache 도입 — 쿼리 튜닝의 구조적 한계 보완"
project: "gathering-server"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Redis
  - Spring Cache
date: 2026-03-11
excerpt: "서브쿼리 개선과 Late Join으로도 해결할 수 없는 구조적 한계를 Redis 캐시로 보완한 과정."
---

## Problem

서브쿼리 개선(2초→200ms)과 Late Join 패턴으로 각 쿼리의 효율성은 크게 향상됐지만, **구조적 한계**가 남아 있었다.

카테고리가 20개면 **20번의 쿼리가 발생**한다. 각 쿼리가 아무리 빨라도, 카테고리 수에 비례하여 쿼리 횟수가 증가하는 구조는 데이터와 트래픽이 늘어날수록 DB 부하로 이어진다.

> "쿼리 튜닝만으로는 데이터 증가에 따른 성능 저하를 완전히 해결할 수 없다."

이 한계를 인식한 것이 캐시 도입의 출발점이었다.

## Action

### Spring Cache + Redis

조회 결과를 Redis에 캐싱하여 **반복적인 DB 조회를 제거**한다.

```java
@Configuration
@EnableCaching
public class CacheConfig {
}
```

```java
@Configuration
public class RedisConfig {

    @Bean
    public LettuceConnectionFactory lettuceConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName(host);
        config.setPort(port);
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(lettuceConnectionFactory());
        return template;
    }
}
```

### 캐시 적용 전략

모임 목록 조회는 **읽기 비율이 압도적으로 높고, 데이터 변경이 상대적으로 드문** 특성을 가진다. 이런 특성은 캐시 도입에 이상적이다.

| 특성 | 값 | 캐시 적합성 |
|------|---|------------|
| 읽기:쓰기 비율 | 약 100:1 | 높음 |
| 데이터 변경 빈도 | 모임 생성/수정 시 | 낮음 |
| 허용 가능한 지연 | 수 분 | 높음 |

### 최적화 단계별 효과 종합

```
Step 1: EXPLAIN ANALYZE 기반 서브쿼리 구조 개선
  → 2초 → 200ms (인덱싱 활용)

Step 2: ROW_NUMBER() → 카테고리별 Late Join 전환
  → 조인 대상 최소화 (100만 건 → 9건)

Step 3: Redis Cache 도입
  → DB 부하 제거 + 응답 속도 일관성 확보
```

## Result

- 쿼리 인덱싱으로 **2초 → 200ms** 개선
- Late Join + Redis 캐시로 **DB 부하 제거**
- 데이터 증가에도 **일관된 응답 속도** 확보
- 캐시 만료 정책으로 **데이터 정합성** 유지

## Reflection

쿼리 튜닝의 3단계를 거치면서 배운 것:

1. **EXPLAIN ANALYZE가 출발점이다** — 느린 쿼리의 원인은 추측이 아니라 실행 계획에서 찾아야 한다
2. **구조를 개선하면 인덱스가 살아난다** — 서브쿼리 내 불필요한 조인을 제거하자 인덱스가 활용되기 시작했다
3. **쿼리 튜닝에도 한계가 있다** — Late Join으로 각 쿼리를 최적화해도, 카테고리 수만큼 쿼리가 발생하는 근본적 한계는 쿼리 레벨에서 해결할 수 없다
4. **한계를 인식하고 레이어를 추가하는 판단이 중요하다** — 캐시는 "쿼리를 더 빠르게" 하는 것이 아니라 "쿼리를 하지 않는" 것이다

문제의 근본 원인을 파악하고, 구조로 해결하는 것. Outbox Pattern, Late Join, EXPLAIN ANALYZE는 그 과정의 결과였다.
