---
title: "TempPayment → Payment 전환 패턴과 트랜잭션 전파 전략"
project: "trip-planner-ai"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - JPA
  - Transaction
date: 2026-03-11
excerpt: "외부 PG 호출과 DB 저장을 트랜잭션 전파 전략으로 분리하고, TempPayment → Payment 전환 패턴으로 결제 유실을 구조적으로 감지하는 설계."
---

## Problem

결제 흐름에서 두 가지 구조적 문제가 있었다.

**1. 외부 API 호출 중 DB 커넥션 점유**

`@Transactional` 기본값(REQUIRED)으로 confirm 메서드를 감싸면, 토스페이먼츠 API 호출(수 초) 동안 DB 커넥션을 점유한다. 동시 결제 요청이 몰리면 **커넥션 풀이 고갈**되어 전체 서비스가 마비된다.

**2. PG 승인 후 DB 저장 실패 시 감지 불가**

Payment를 바로 생성하는 구조에서는, PG 승인은 성공했지만 DB 저장이 실패하면 **결제 유실을 감지할 방법이 없다**. 유저의 돈은 빠져나갔는데 서비스에는 기록이 없는 상태가 된다.

## Action

![트랜잭션 전파 전략](/assets/images/trip-payment/transaction-propagation.png)

### NOT_SUPPORTED — 외부 API 호출 구간

`PaymentService.confirm()`에 `Propagation.NOT_SUPPORTED`를 적용했다. 이 전파 속성은 **기존 트랜잭션을 일시 중단**하고, 트랜잭션 없이 메서드를 실행한다.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public ApiResponse confirm(String idempotencyKey, PaymentRequest paymentRequest, Long id) {
    // Redis 멱등성 체크 — 트랜잭션 불필요
    Boolean success = redisTemplate.opsForValue()
        .setIfAbsent(idempotencyKey, "true", Duration.of(24L, ChronoUnit.HOURS));
    ...

    // DB 멱등성 체크 — 트랜잭션 불필요 (저장은 별도)
    idempotencyRepository.save(Idempotency.of(idempotencyKey));

    // 외부 PG API 호출 — DB 커넥션 점유 X
    boolean confirm = paymentClient.confirm(paymentRequest);

    if (confirm) {
        // DB 저장은 별도 트랜잭션으로 위임
        paymentFacade.processConfirm(paymentRequest, id);
    }
    return ApiStatusResponse.of(Status.SUCCESS);
}
```

외부 PG API 호출이 3초 걸려도, 그 동안 DB 커넥션을 점유하지 않는다.

### REQUIRES_NEW — DB 저장 구간

`PaymentFacade`에 `Propagation.REQUIRES_NEW`를 적용했다. 부모 트랜잭션과 무관하게 **독립적인 새 트랜잭션**을 시작한다.

```java
@Component
@Transactional(propagation = Propagation.REQUIRES_NEW)
public class PaymentFacade {

    public void processConfirm(PaymentRequest paymentRequest, Long id) {
        String paymentKey = paymentRequest.getPaymentKey();
        String orderId = paymentRequest.getOrderId();
        Long amount = paymentRequest.getAmount();

        Member member = memberRepository.findById(id)
            .orElseThrow(() -> new CommonException(Status.NOT_FOUND_MEMBER));

        // 1. TempPayment 조회 (PENDING 상태)
        TempPayment tempPayment = tempPaymentRepository
            .findByPaymentKeyAndOrderIdAndAmountAndStatus(
                paymentKey, orderId, amount, PaymentStatus.PENDING)
            .orElseThrow(() -> new CommonException(Status.NOT_FOUND_PAYMENT));

        // 2. Payment 생성 (COMPLETE 상태)
        Payment payment = PaymentFactory.from(paymentRequest, member);
        paymentRepository.save(payment);

        // 3. 포인트 적립
        Point point = pointRepository.findByMember(member)
            .orElseThrow(() -> new CommonException(Status.NOT_FOUND_POINT));
        point.addAmount(amount);

        // 4. TempPayment 삭제
        tempPaymentRepository.delete(tempPayment);
    }
}
```

이 메서드 안에서 예외가 발생하면, Payment 저장·포인트 적립·TempPayment 삭제가 **전부 롤백**된다. 원자성이 보장되므로 중간 상태가 존재하지 않는다.

### TempPayment → Payment 전환 패턴

이 설계의 핵심은 **결제 의도와 결제 완료를 분리**한 것이다.

| 엔티티 | 역할 | 상태 |
|--------|------|------|
| TempPayment | 결제 의도 기록 | PENDING |
| Payment | 결제 완료 기록 | COMPLETE |

```
클라이언트 결제 요청
  → TempPayment 생성 (PENDING)
    → PG 승인 + 멱등성 체크
      → Payment 생성 (COMPLETE)
        → TempPayment 삭제
```

**왜 TempPayment가 필요한가?**

Payment를 바로 생성하는 구조에서는, PG 승인 성공 후 DB 저장이 실패하면 결제 유실을 **감지할 수 없다**. 하지만 TempPayment가 남아 있으면:

- Reconciliation 스케줄러가 PENDING 상태의 TempPayment를 발견
- PG에 직접 조회하여 승인 여부 확인
- 승인 완료 상태면 Payment를 생성하여 복구

TempPayment는 **"이 결제가 아직 완료되지 않았다"는 신호**를 시스템에 남기는 역할을 한다.

### 트랜잭션 전파 전략 요약

| 컴포넌트 | 전파 속성 | 이유 |
|----------|-----------|------|
| PaymentService.confirm() | NOT_SUPPORTED | 외부 PG 호출 시 DB 커넥션 점유 방지 |
| PaymentService.cancel() | NOT_SUPPORTED | 동일 이유 |
| PaymentFacade | REQUIRES_NEW | DB 저장 원자성 보장, 독립 롤백 |

## Result

- 외부 PG API 호출과 DB 저장을 **트랜잭션 레벨에서 완전히 분리**
- `NOT_SUPPORTED`로 PG 호출 중 커넥션 풀 고갈 방지
- `REQUIRES_NEW`로 Payment 저장·포인트 적립·TempPayment 삭제의 **원자성 보장**
- TempPayment → Payment 전환 패턴으로 **결제 유실을 구조적으로 감지** 가능

## Reflection

`@Transactional`은 편리하지만, 기본값(REQUIRED)만으로는 외부 API 호출이 포함된 결제 흐름을 안전하게 처리할 수 없다. 트랜잭션 경계를 의도적으로 설계해야 한다.

TempPayment → Payment 전환 패턴은 단순한 상태 변경이 아니라, **"아직 완료되지 않은 결제"를 시스템이 인식할 수 있게 하는 구조적 장치**다. 이 패턴이 없었다면 3중 안전망의 Reconciliation 스케줄러는 동작할 수 없었을 것이다.

결국 트랜잭션 전파 전략(NOT_SUPPORTED + REQUIRES_NEW)과 TempPayment → Payment 전환 패턴은 독립적인 기술이 아니라, **결제 안전성이라는 하나의 목표를 위해 서로 맞물려 동작하는 설계**였다.
