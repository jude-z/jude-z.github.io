---
title: "결제 테스트 3단계 — PG 승인 후 DB 저장 실패, Webhook이 살릴 수 있는가"
project: "trip-planner-ai"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Redis
  - Payment
  - Webhook
date: 2026-03-12
excerpt: "PG 승인은 성공했지만 DB 저장을 의도적으로 스킵한 뒤, PG Webhook이 결제를 복구하는 과정을 테스트로 검증한다."
---

## Problem

[2단계](/projects/trip-payment-v2-redis-idempotency/)에서 Redis 멱등성이 정상 시 동작하는 것을 확인했다. 하지만 결제 흐름에는 또 다른 위험 구간이 존재한다.

```
클라이언트 → 서버 → Redis 체크 ✓ → PG 승인 ✓ → DB 저장 ???
```

PG 승인까지 성공한 시점에서 **서버가 죽거나 DB 커넥션이 끊기면**, Payment가 저장되지 않는다. 유저의 돈은 빠져나갔는데 서비스에는 기록이 없는 **결제 유실**이 발생한다.

토스페이먼츠는 결제 승인 후 **Webhook으로 결과를 통보**한다. 이 Webhook이 결제 유실을 복구할 수 있는지 테스트해야 한다.

> "PG 승인은 됐는데 DB 저장이 실패하면, Webhook만으로 복구가 되는가?"

이 질문에 답하기 위해, **의도적으로 DB 저장을 스킵**하고 Webhook 복구를 검증했다.

## Action

### 시스템 아키텍처 — DB 스킵 + Webhook 복구 흐름

![V3 아키텍처](/assets/images/trip-payment/v3-webhook-recovery.png)

### V3 코드: 의도적 DB 저장 스킵

```java
public ApiResponse confirmNoDbSave(String idempotencyKey,
                                    PaymentRequest paymentRequest, Long id) {
    if (!idempotencyRedisManager.tryAcquire(idempotencyKey)) {
        throw new CommonException(Status.ALREADY_PAYMENT_REQUEST);
    }

    log.warn("[NO_DB_SAVE] DB 멱등성 저장 스킵 - 웹훅으로 보완 필요. key={}", idempotencyKey);

    boolean confirm = paymentClient.confirm(paymentRequest);
    if (confirm) {
        // 핵심: processConfirm()을 호출하지 않는다
        // → Payment 미생성, 포인트 미적립, TempPayment 미삭제
        idempotencyRedisManager.complete(idempotencyKey);
    } else {
        idempotencyRedisManager.fail(idempotencyKey);
    }
    return ApiStatusResponse.of(Status.SUCCESS);
}
```

`paymentFacade.processConfirm()`을 **의도적으로 호출하지 않는다**. PG 승인은 성공하지만:
- Payment 레코드 **미생성**
- 포인트 **미적립**
- TempPayment는 **PENDING 상태 그대로**

실제 운영에서 DB 장애가 발생한 상황을 시뮬레이션한 것이다.

### Webhook 복구 로직: WebHookService

PG가 결제 완료를 Webhook으로 통보하면, `WebHookService`가 복구를 시도한다.

```java
@Service
public class WebHookService {

    public void processWebHook(JsonNode jsonNode) {
        JsonNode data = jsonNode.get("data");
        String status = data.get("status").asText();
        String paymentKey = data.get("paymentKey").asText();
        String orderId = data.get("orderId").asText();
        long amount = Long.parseLong(data.get("totalAmount").asText());

        // 1. DONE이 아닌 상태는 무시
        if (!status.equals("DONE")) {
            webHookHistoryRepository.save(
                WebHookHistory.of(paymentKey, orderId, amount, status, false));
            return;
        }

        // 2. 이미 처리된 Webhook인지 확인 (멱등성)
        Optional<WebHookHistory> processed =
            webHookHistoryRepository.findByPaymentKeyAndProcessedTrue(paymentKey);
        if (processed.isPresent()) return;

        // 3. TempPayment 조회 (결제 의도 확인)
        TempPayment tempPayment = tempPaymentRepository
            .findByPaymentKeyAndOrderIdAndAmountAndStatus(
                paymentKey, orderId, amount, PaymentStatus.PENDING)
            .orElseThrow(() -> new CommonException(Status.NOT_FOUND_TEMP_PAYMENT));

        // 4. Payment가 이미 존재하는지 확인 (confirm 경로에서 생성됐을 수 있음)
        Optional<Payment> existingPayment =
            paymentRepository.findByPaymentKeyAndOrderIdAndAmount(
                paymentKey, orderId, amount);

        // 5. Payment 미존재 시 복구
        if (existingPayment.isEmpty()) {
            Member member = memberRepository.findById(tempPayment.getMember().getId())
                .orElseThrow(() -> new CommonException(Status.NOT_FOUND_MEMBER));

            paymentRepository.save(PaymentFactory.from(paymentKey, orderId, amount, member));
            tempPaymentRepository.delete(tempPayment);

            Point point = pointRepository.findByMember(member)
                .orElseThrow(() -> new CommonException(Status.NOT_FOUND_POINT));
            point.addAmount(amount);
        }

        // 6. 처리 완료 기록
        webHookHistoryRepository.save(
            WebHookHistory.of(paymentKey, orderId, amount, status, true));
    }
}
```

Webhook의 멱등성은 `WebHookHistory.processed` 필드로 보장한다. 동일한 Webhook이 여러 번 들어와도 **processed=true인 레코드가 존재하면 무시**한다.

복구 흐름:
1. DONE 상태 확인
2. WebHookHistory로 중복 처리 방지
3. TempPayment(PENDING)로 결제 의도 확인
4. Payment가 없으면 생성 + 포인트 적립 + TempPayment 삭제
5. WebHookHistory에 processed=true 기록

## Result

### confirm 직후: Payment 0건

![V3 PG서버 DB](/assets/images/trip-payment/v3%20pg%20.png)

mock PG 서버에는 1건 저장. Redis 멱등성이 정상 동작하여 PG에는 1건만 도달했다.

![V3 TempPayment](/assets/images/trip-payment/v3%20temppayment.png)

`temp_payment` 테이블에 1건이 **PENDING 상태**로 남아 있다. processConfirm()을 스킵했기 때문에 TempPayment가 삭제되지 않았다.

![V3 Payment 이전](/assets/images/trip-payment/v3%20이전%20payment.png)

`payment` 테이블은 **0건**. DB 저장이 스킵됐으므로 결제 완료 레코드가 없다.

이 시점에서의 상태:
- **PG**: 승인 완료 (돈은 빠져나감)
- **서비스 DB**: 결제 기록 없음
- **TempPayment**: PENDING (결제 의도만 남아 있음)

**결제 유실 상태다.**

### Webhook 수신 후: Payment 1건 복구

![V3 Payment 복구](/assets/images/trip-payment/v3%20이후.png)

PG Webhook을 수신한 후, `payment` 테이블에 **1건이 COMPLETE 상태**로 생성됐다. 포인트도 10,000원이 정상 적립됐다.

| 시점 | Payment | 포인트 | TempPayment |
|------|---------|--------|-------------|
| confirm 직후 | 0건 | 0원 | PENDING |
| **Webhook 후** | **1건 COMPLETE** | **10,000원** | 삭제됨 |

Webhook이 결제 유실을 **완전히 복구**했다.

## Reflection

이 테스트가 증명한 것은 **Webhook 단독으로도 결제 유실을 복구할 수 있다**는 것이다. TempPayment가 PENDING 상태로 남아 있는 한, Webhook은 결제 의도를 확인하고 Payment를 생성할 수 있다.

하지만 Webhook에는 구조적 한계가 있다:

1. **PG 서버가 Webhook을 보내지 못할 수 있다** — PG 장애, 네트워크 단절
2. **우리 서버가 Webhook을 수신하지 못할 수 있다** — 서버 다운, 네트워크 이슈
3. **Webhook은 PG가 주도한다** — 우리가 재시도 시점을 제어할 수 없다

Webhook은 "2층 안전망"이지만, 최종 보정은 **우리 서버가 직접 PG에 조회하는 구조**가 필요하다. Webhook에 의존하지 않고, 스케줄러가 PENDING 상태의 TempPayment를 주기적으로 확인하여 PG에 직접 물어보는 것이다.

---

> Webhook으로 복구는 가능하다. 하지만 Webhook마저 안 오면? 최종 보정을 우리가 직접 해야 한다.
> [4단계: Redis + DB 이중 멱등성 + Reconciliation 스케줄러 →](/projects/trip-payment-v4-full-safety/)
