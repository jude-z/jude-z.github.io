---
title: "결제 테스트 2단계 — Redis 멱등성은 동작하지만, Redis가 죽으면?"
project: "trip-planner-ai"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Redis
  - Payment
  - Concurrency
date: 2026-03-12
excerpt: "Redis setIfAbsent로 동시 결제를 차단하는 데 성공했지만, Redis 장애 시 멱등성이 뚫리는 SPOF 문제를 테스트로 증명한다."
---

## Problem

[1단계](/projects/trip-payment-v1-no-idempotency/)에서 멱등성 없이 100개의 스레드를 보내면 PG에 100건이 승인되는 것을 확인했다. 해결책은 명확하다 — **첫 번째 요청만 PG에 전달하고, 나머지는 차단**해야 한다.

Redis의 `setIfAbsent`(SETNX)는 원자적 연산이다. 동시에 100개의 요청이 들어와도 **딱 하나만 true를 반환**한다. Redis 단일 스레드 특성상 경쟁 조건이 발생하지 않는다.

하지만 질문이 남는다:
> "Redis가 정상일 때는 완벽하다. 그런데 Redis가 내려가면?"

이 테스트는 두 가지를 검증한다:
1. Redis가 정상일 때 멱등성이 동작하는가
2. Redis가 장애일 때 무슨 일이 벌어지는가

## Action

### 시스템 아키텍처 — Redis 멱등성과 장애 시나리오

![V2 시스템 아키텍처 — Redis 멱등성 정상 동작과 SPOF 문제](/assets/images/trip-payment/v2-redis-idempotency.png)

### V2 코드: Redis 멱등성 + 장애 시 스킵

```java
public ApiResponse confirmRedisFail(String idempotencyKey,
                                     PaymentRequest paymentRequest, Long id) {
    try {
        if (!idempotencyRedisManager.tryAcquire(idempotencyKey)) {
            throw new CommonException(Status.ALREADY_PAYMENT_REQUEST);
        }
    } catch (CommonException e) {
        throw e;  // 이미 처리된 요청 — 그대로 거부
    } catch (Exception e) {
        // Redis 장애 시 — 멱등성 체크를 건너뛴다
        log.warn("[V2] Redis 장애 발생 - 멱등성 체크 스킵. key={}", idempotencyKey);
    }

    boolean confirm = paymentClient.confirm(paymentRequest);
    if (confirm) {
        paymentFacade.processConfirm(paymentRequest, id);
    }
    return ApiStatusResponse.of(Status.SUCCESS);
}
```

핵심은 `catch (Exception e)` 블록이다. Redis 연결 실패, 타임아웃 등 Redis 관련 예외가 발생하면 **경고 로그만 남기고 결제를 진행**한다. "Redis가 죽었으니 결제도 안 되면 안 되지 않나?"라는 판단이다.

### IdempotencyRedisManager — 원자적 락 획득

```java
public boolean tryAcquire(String key) {
    Boolean result = redisTemplate.opsForValue()
        .setIfAbsent(key, IdempotencyStatus.PROCESSING.name(),
                     Duration.ofHours(24));
    if (result == null) {
        throw new CommonException(Status.REDIS_SERVER_ERROR);
    }
    return result;  // true: 첫 요청, false: 중복 요청
}

public void complete(String key) {
    redisTemplate.opsForValue().set(key, IdempotencyStatus.COMPLETED.name());
}

public void fail(String key) {
    redisTemplate.delete(key);  // 키 삭제 → 재시도 허용
}
```

- `tryAcquire()`: `SETNX` + 24시간 TTL. 첫 요청만 true.
- `complete()`: 성공 시 상태를 COMPLETED로 변경
- `fail()`: 실패 시 키를 삭제하여 재시도를 허용

### 테스트 1: Redis 정상 — 100개의 스레드 동시 요청

```java
@Test
@DisplayName("V2: Redis 정상 — 동시 100건 중 1건만 PG 호출")
void v2_redisNormal_concurrentRequests() throws InterruptedException {
    int threadCount = 100;
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
    CountDownLatch latch = new CountDownLatch(threadCount);

    for (int i = 0; i < threadCount; i++) {
        executorService.submit(() -> {
            try {
                sendConfirmRequest("/api/v2/confirm", body, "paymentKey1");
            } finally {
                latch.countDown();
            }
        });
    }
    latch.await();
}
```

100개의 스레드가 동일한 `Idempotency-Key: paymentKey1`로 동시 요청한다.

### 테스트 2: Redis 장애 시뮬레이션

```java
@Test
@DisplayName("V2: Redis 장애 — 멱등성 뚫림 확인")
void v2_redisFailure_concurrentRequests() throws InterruptedException {
    // Phase 1: Redis 정상 — 50개의 스레드
    // → 1건만 PG 도달

    Thread.sleep(10000);  // Redis 수동 중지

    // Phase 2: Redis DOWN — 50개의 스레드
    // → catch 블록에서 스킵 → 전부 PG 도달
}
```

Phase 1에서 50개의 스레드를 보내고, Redis를 수동으로 중지한 후 Phase 2에서 50개의 스레드를 추가로 보낸다.

## Result

### 테스트 1 결과: Redis 정상

![V2 PG서버 DB](/assets/images/trip-payment/v2%20pg서버%20db%20저장내역.png)

mock PG 서버에 **1건만 저장**. Redis `setIfAbsent`가 99개의 중복 요청을 완벽하게 차단했다.

![V2 실제 DB](/assets/images/trip-payment/v2%20실제%20db%20저장내역.png)

실제 DB의 `point` 테이블에도 정상적으로 10,000원이 적립됐다. V1에서 발생했던 데이터 불일치가 **완전히 해소**됐다.

| 저장소 | V1 (멱등성 없음) | V2 (Redis 정상) |
|--------|-----------------|----------------|
| PG 서버 | 100건 | **1건** |
| 실제 DB | 1건 | **1건** |
| 포인트 | 불확실 | **10,000원 정상** |

### 테스트 2 결과: Redis 장애

![V2 Redis 장애 테스트](/assets/images/trip-payment/v2%20실제%20테스트%20내역.png)

Redis가 내려간 Phase 2에서는:
- `[V2] Redis 장애 발생 - 멱등성 체크 스킵` 경고 로그 출력
- 모든 요청이 `catch (Exception e)` 블록을 타고 **PG까지 도달**
- 성공 1건, 실패 99건 (V1과 동일한 패턴)

**결론: Redis가 죽으면 V1과 동일한 상황이 재현된다.** Redis가 단일 장애점(SPOF)이 된 것이다.

### 왜 catch에서 결제를 진행하는가?

"Redis 장애 시 결제를 차단하면 되지 않나?"라는 의문이 들 수 있다. 하지만 **Redis 장애 때문에 결제 자체가 불가능해지면**, 그것은 또 다른 장애다. 가용성과 정합성 사이의 트레이드오프에서, V2는 **가용성을 선택하고 정합성을 희생**한 것이다.

## Reflection

V2가 증명한 것은 두 가지다:

**1. Redis 멱등성은 동작한다.** setIfAbsent의 원자성 덕분에 100개의 동시 요청 중 단 1건만 통과시키는 것이 가능하다. 정상 상황에서는 완벽한 해결책이다.

**2. Redis는 단일 장애점이 된다.** Redis에 의존하는 모든 보호 장치는 Redis와 함께 사라진다. catch 블록에서 "일단 진행"하는 판단은 가용성을 지키지만, 결제 정합성이라는 핵심 가치를 포기한 것이다.

결제 시스템에서 정합성을 포기하는 것은 허용되지 않는다. Redis가 죽어도 멱등성이 보장되려면, **Redis가 아닌 다른 곳에도 멱등성 상태를 저장**해야 한다. 그것이 DB다.

하지만 그 전에 먼저 검증할 것이 있다 — PG 승인까지 성공했는데 **DB 저장이 실패하면?** 이 시나리오를 의도적으로 만들어 Webhook이 복구할 수 있는지 테스트해야 한다.

---

> Redis 멱등성은 정상 시 완벽하지만 SPOF 문제가 있다. 그런데 더 근본적인 질문 — PG 승인 후 DB 저장 자체가 실패하면?
> [3단계: DB 저장 실패 시 Webhook이 복구할 수 있는가 →](/projects/trip-payment-v3-webhook-recovery/)
