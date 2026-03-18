---
title: "결제 멱등성 설계 — Redis + DB 이중 구조로 중복 결제 원천 차단"
project: "trip-planner-ai"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Redis
  - Payment
date: 2026-02-25
excerpt: "클라이언트 재시도 시 동일한 결제가 두 번 승인되는 문제를 Redis setIfAbsent + DB Idempotency 이중 구조로 해결한 과정."
---

## Problem

토스페이먼츠 결제 연동에서 클라이언트가 네트워크 타임아웃 등으로 **동일한 결제 요청을 재시도**하면, PG사에 승인이 두 번 들어가는 중복 결제 문제가 존재한다.

단순히 Redis로 멱등성을 처리하면 되지 않냐는 생각이 들 수 있지만, **Redis가 내려가면 멱등성 체크 자체가 불가능**해진다. Redis 단일 장애점(SPOF)을 그대로 두면, 장애 상황에서 오히려 중복 결제가 발생하는 최악의 시나리오가 된다.

질문은 단순했다:
> "Redis가 죽어도 중복 결제가 발생하지 않으려면 어떻게 해야 하는가?"

## Action

![결제 멱등성 아키텍처](/assets/images/trip-payment/idempotency-architecture.png)

### Redis setIfAbsent — 1차 멱등성 체크 (속도)

클라이언트가 보내는 `Idempotency-Key` 헤더를 Redis에 `SETNX(setIfAbsent)`로 저장한다. 이미 키가 존재하면 중복 요청으로 판단하여 즉시 차단한다.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public ApiResponse confirm(String idempotencyKey, PaymentRequest paymentRequest, Long id) {

    // 1차: Redis 멱등성 체크 (원자적 연산)
    Boolean success = redisTemplate.opsForValue()
        .setIfAbsent(idempotencyKey, "true", Duration.of(24L, ChronoUnit.HOURS));

    if (success == null) throw new CommonException(Status.REDIS_SERVER_ERROR);
    if (!success) throw new CommonException(Status.ALREADY_PAYMENT_REQUEST);

    // 2차: DB 멱등성 체크 (Redis 장애 대비)
    Optional<Idempotency> optionalIdempotency =
        queryDslIdempotencyRepository.findByIdempotencyKey(idempotencyKey);
    if (optionalIdempotency.isPresent()) {
        throw new CommonException(Status.ALREADY_PAYMENT_REQUEST);
    }
    idempotencyRepository.save(Idempotency.of(idempotencyKey));

    // PG 승인 요청
    boolean confirm = paymentClient.confirm(paymentRequest);
    if (confirm) {
        paymentFacade.processConfirm(paymentRequest, id);
    }
    return ApiStatusResponse.of(Status.SUCCESS);
}
```

핵심은 `setIfAbsent`의 **원자성**이다. 동시에 10개의 재시도 요청이 들어와도, Redis의 단일 스레드 특성상 딱 하나의 요청만 `true`를 반환받는다.

### DB Idempotency 테이블 — 2차 멱등성 체크 (영속성)

Redis가 정상일 때는 1차에서 걸러지지만, **Redis가 내려간 상황**에서는 DB가 멱등성을 보장한다. `Idempotency` 테이블에 동일 키를 UNIQUE 제약조건으로 저장하여, DB 레벨에서도 중복을 원천 차단한다.

```
Redis(속도) + DB(영속성) = 단일 장애점 없는 멱등성 이중 보장
```

### 왜 Propagation.NOT_SUPPORTED인가?

`confirm()` 메서드에 `NOT_SUPPORTED`를 적용한 이유는, **외부 PG API 호출 시 DB 커넥션을 점유하지 않기 위해서**다. 토스페이먼츠 API 응답이 수 초 걸릴 수 있는데, 그 동안 DB 트랜잭션을 열어두면 커넥션 풀이 고갈된다.

실제 DB 저장은 `PaymentFacade`가 `REQUIRES_NEW`로 독립 트랜잭션에서 처리한다:

```java
@Component
@Transactional(propagation = Propagation.REQUIRES_NEW)
public class PaymentFacade {

    public void processConfirm(PaymentRequest paymentRequest, Long id) {
        TempPayment tempPayment = tempPaymentRepository
            .findByPaymentKeyAndOrderIdAndAmountAndStatus(
                paymentKey, orderId, amount, PaymentStatus.PENDING)
            .orElseThrow(() -> new CommonException(Status.NOT_FOUND_PAYMENT));

        Payment payment = PaymentFactory.from(paymentRequest, member);
        paymentRepository.save(payment);

        point.addAmount(amount);
        tempPaymentRepository.delete(tempPayment);
    }
}
```

## Result

- Redis(속도) + DB(영속성) 이중 구조로 **단일 장애점 없는 멱등성** 보장
- `setIfAbsent` 원자적 연산으로 동시 요청 차단
- `NOT_SUPPORTED` + `REQUIRES_NEW` 트랜잭션 전파로 **외부 API 호출과 DB 저장을 분리**

## Reflection

멱등성을 단순히 "중복 방지"로만 생각했다면, Redis 하나로 끝났을 것이다. 하지만 결제는 **장애 시나리오를 역추적**해야 한다. "Redis가 죽으면?"이라는 질문 하나가 DB 이중 구조의 출발점이었다.

트랜잭션 전파 전략도 마찬가지다. `@Transactional`을 기본값으로 두면 동작은 하지만, 외부 API 호출이 포함된 메서드에서 DB 커넥션을 수 초간 점유하는 문제를 인식하지 못하면 운영 환경에서 커넥션 풀 고갈이라는 장애로 돌아온다.
