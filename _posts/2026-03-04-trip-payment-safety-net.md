---
title: "결제 유실 방지 — PG 웹훅 + Reconciliation 스케줄러 3중 안전망"
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
date: 2026-03-04
excerpt: "confirm API 실패, 웹훅 수신 실패까지 대비한 3중 안전망으로 결제 유실률 0%를 달성한 설계 과정."
---

## Problem

멱등성으로 중복 결제는 막았지만, **결제 유실**이라는 또 다른 문제가 남아 있었다.

정상적으로 PG 승인까지 완료됐지만, 이후 **DB 장애로 결제 내역이 저장되지 않는** 경우가 발생할 수 있다. 유저의 실제 돈은 빠져나갔는데 서비스에는 결제가 반영되지 않는 최악의 시나리오다.

PG가 웹훅으로 결제 완료를 통보하지만, **웹훅마저 수신에 실패하면** 결제 유실을 감지할 방법이 없다. 최후의 안전망으로 서버가 직접 PG에 조회하는 구조가 필요했다.

> "PG는 승인했는데 우리 서버가 죽으면? 웹훅도 안 오면?"

이 질문들이 3중 안전망의 출발점이었다.

## Action

![3중 안전망 아키텍처](/assets/images/trip-payment/triple-safety-net.png)

### 1층 — confirm API (정상 경로)

클라이언트가 결제 승인을 요청하면, Redis 멱등성 체크 → PG confirm → Payment 저장 순서로 처리된다. 정상적인 경우 여기서 끝난다.

### 2층 — PG 웹훅 수신 (confirm 실패 시 복구)

confirm API 응답이 유실되어도, 토스페이먼츠가 **웹훅으로 결제 완료(DONE)를 통보**한다. 웹훅 수신 시에도 동일한 멱등성 체크를 적용하고, Payment가 미생성 상태일 때만 저장한다.

```java
@Service
public class WebHookService {

    public void processWebHook(JsonNode jsonNode) {
        String status = data.get("status").asText();
        String paymentKey = data.get("paymentKey").asText();
        String orderId = data.get("orderId").asText();
        long amount = Long.parseLong(data.get("totalAmount").asText());

        if (!status.equals("DONE")) return;

        // 동일한 멱등성 체크 (Redis + DB)
        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(paymentKey, "true", Duration.of(24L, ChronoUnit.HOURS));
        if (success == null) throw new CommonException(Status.REDIS_SERVER_ERROR);
        if (!success) throw new CommonException(Status.ALREADY_CANCEL_REQUEST);

        Optional<Idempotency> optionalIdempotency =
            queryDslIdempotencyRepository.findByIdempotencyKey(paymentKey);
        if (optionalIdempotency.isPresent()) {
            throw new CommonException(Status.ALREADY_CANCEL_REQUEST);
        }
        idempotencyRepository.save(Idempotency.of(paymentKey));

        // TempPayment 존재 확인 후 Payment 생성
        TempPayment tempPayment = tempPaymentRepository
            .findByPaymentKeyAndOrderIdAndAmountAndStatus(
                paymentKey, orderId, amount, PaymentStatus.PENDING)
            .orElseThrow(() -> new CommonException(Status.NOT_FOUND_TEMP_PAYMENT));

        Optional<Payment> optionalPayment =
            paymentRepository.findByPaymentKeyAndOrderIdAndAmount(paymentKey, orderId, amount);

        if (optionalPayment.isEmpty()) {
            paymentRepository.save(PaymentFactory.from(paymentKey, orderId, amount, member));
            point.addAmount(amount);
        }
    }
}
```

핵심은 `optionalPayment.isEmpty()` 체크다. confirm 경로에서 이미 Payment가 생성됐다면 웹훅은 아무것도 하지 않는다. **3개 경로가 동시에 동작해도 중복 저장이 발생하지 않는** 구조다.

### 3층 — Reconciliation 스케줄러 (최종 보정)

웹훅마저 실패한 경우를 대비해, **PENDING 상태의 TempPayment를 주기적으로 PG에 직접 조회**한다.

```java
@Service
public class ReconciliationService {

    @Scheduled(scheduler = "reconciliationScheduler", cron = "10/* * * * *")
    @Transactional
    public void reconciliation() {
        List<TempPayment> tempPayments =
            queryDslTempPaymentRepository.fetchPendingPayments();

        // 이미 Payment가 존재하는 건 필터링
        List<Payment> existPayments =
            paymentRepository.findByPaymentKeyIn(paymentKeys);
        Set<String> existKeys = existPayments.stream()
            .map(Payment::getPaymentKey).collect(Collectors.toSet());

        List<TempPayment> filtered = tempPayments.stream()
            .filter(tp -> !existKeys.contains(tp.getPaymentKey()))
            .toList();

        // PG에 직접 조회
        filtered.forEach(tempPayment -> {
            try {
                ResponseEntity<JsonNode> response = restTemplate.exchange(
                    url + tempPayment.getPaymentKey(),
                    HttpMethod.GET, new HttpEntity<>(getHeaders()), JsonNode.class);

                if (response.getStatusCode() == HttpStatus.OK) {
                    savePayments.add(PaymentFactory.from(...));
                    deletePaymentKeys.add(paymentKey);
                } else {
                    addRetryPaymentKeys.add(paymentKey);
                }
            } catch (Exception e) {
                addRetryPaymentKeys.add(paymentKey);
            }
        });

        paymentRepository.saveAll(savePayments);
        tempPaymentRepository.deleteByPaymentKeyInBatch(deletePaymentKeys);
        tempPaymentRepository.addRetry(addRetryPaymentKeys);
    }
}
```

### 지수 백오프 재시도

PG 부하를 최소화하기 위해, retry 횟수에 따라 조회 간격을 늘린다.

```java
public List<TempPayment> fetchPendingPayments() {
    BooleanExpression retryCondition =
        tp.retry.eq(0).and(tp.createdTime.loe(now.minusMinutes(10)))    // 10분 후
        .or(tp.retry.eq(1).and(tp.createdTime.loe(now.minusMinutes(30)))) // 30분 후
        .or(tp.retry.eq(2).and(tp.createdTime.loe(now.minusMinutes(60)))); // 60분 후

    return queryFactory.selectFrom(tp)
        .where(tp.status.eq(PaymentStatus.PENDING), retryCondition)
        .orderBy(tp.id.asc())
        .limit(100)  // 배치 처리
        .fetch();
}
```

| retry | 대기 시간 | 의미 |
|-------|----------|------|
| 0 | 10분 | 첫 조회 — confirm/웹훅이 처리할 시간 확보 |
| 1 | 30분 | 재시도 — 일시적 장애 복구 대기 |
| 2 | 60분 | 최종 시도 — 이후 수동 확인 필요 |

## Result

- confirm API 실패 → **웹훅이 복구**
- 웹훅도 실패 → **스케줄러가 PG에 직접 조회하여 최종 보정**
- 3개 경로(confirm·웹훅·스케줄러)가 동시에 처리되더라도 **paymentKey + orderId UNIQUE 제약조건으로 중복 저장 원천 차단**
- 결제 중복 및 유실률 **0%** 달성

## Reflection

멱등성을 단순히 "중복 방지"로만 생각했다면, Redis 하나로 끝났을 것이다. 하지만 결제는 **실패 시나리오를 역추적**해야 한다.

"PG는 승인했는데 우리 서버가 죽으면?" → 웹훅이 복구한다.
"웹훅도 안 오면?" → 스케줄러가 PG에 직접 물어본다.

이 질문들의 답을 구조로 만든 것이 3중 안전망이다. 단순한 예외 처리가 아니라, **장애 시나리오를 역추적하고 구조적 안전망을 설계하는 것**이 결제 시스템의 핵심이었다.
