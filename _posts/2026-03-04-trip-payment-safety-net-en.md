---
title: "Payment Loss Prevention — Triple Safety Net with PG Webhook + Reconciliation"
project: "trip-planner-ai"
lang: en
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Redis
  - Payment
  - Webhook
date: 2026-03-04
excerpt: "Designing a triple safety net — confirm API, PG webhook, and Reconciliation scheduler — to achieve 0% payment loss rate."
---

## Problem

Idempotency solved duplicate payments, but **payment loss** remained as another critical problem.

Even after a successful PG approval, a **DB failure could prevent the payment record from being saved**. The user's money is charged, but the service has no record of it — the worst-case scenario.

The PG sends a webhook notification on payment completion, but **if the webhook delivery also fails**, there's no way to detect the lost payment. A last-resort mechanism where the server directly queries the PG was needed.

> "What if the PG approved it but our server died? What if the webhook never arrives?"

These questions were the starting point for the triple safety net.

## Action

![Triple Safety Net Architecture](/assets/images/trip-payment/triple-safety-net.png)

### Layer 1 — confirm API (Normal Path)

When the client requests payment approval: Redis idempotency check → PG confirm → Payment save. In normal cases, it ends here.

### Layer 2 — PG Webhook (Recovery on confirm Failure)

Even if the confirm API response is lost, Toss Payments **sends a webhook notification with payment status DONE**. The webhook handler applies the same idempotency check and only saves Payment when it doesn't already exist.

```java
@Service
public class WebHookService {

    public void processWebHook(JsonNode jsonNode) {
        String status = data.get("status").asText();
        String paymentKey = data.get("paymentKey").asText();
        String orderId = data.get("orderId").asText();
        long amount = Long.parseLong(data.get("totalAmount").asText());

        if (!status.equals("DONE")) return;

        // Same idempotency check (Redis + DB)
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

        // Check TempPayment exists, then create Payment
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

The key is the `optionalPayment.isEmpty()` check. If Payment was already created via the confirm path, the webhook does nothing. **Even when all 3 paths run concurrently, no duplicate records are created.**

### Layer 3 — Reconciliation Scheduler (Final Correction)

For cases where even the webhook fails, the scheduler **periodically queries the PG directly for PENDING TempPayments**.

```java
@Service
public class ReconciliationService {

    @Scheduled(scheduler = "reconciliationScheduler", cron = "10/* * * * *")
    @Transactional
    public void reconciliation() {
        List<TempPayment> tempPayments =
            queryDslTempPaymentRepository.fetchPendingPayments();

        // Filter out those already saved as Payment
        Set<String> existKeys = paymentRepository.findByPaymentKeyIn(paymentKeys)
            .stream().map(Payment::getPaymentKey).collect(Collectors.toSet());

        List<TempPayment> filtered = tempPayments.stream()
            .filter(tp -> !existKeys.contains(tp.getPaymentKey())).toList();

        // Direct PG query
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

### Exponential Backoff Retry

To minimize PG load, the query interval increases with retry count:

```java
public List<TempPayment> fetchPendingPayments() {
    BooleanExpression retryCondition =
        tp.retry.eq(0).and(tp.createdTime.loe(now.minusMinutes(10)))    // after 10 min
        .or(tp.retry.eq(1).and(tp.createdTime.loe(now.minusMinutes(30)))) // after 30 min
        .or(tp.retry.eq(2).and(tp.createdTime.loe(now.minusMinutes(60)))); // after 60 min

    return queryFactory.selectFrom(tp)
        .where(tp.status.eq(PaymentStatus.PENDING), retryCondition)
        .orderBy(tp.id.asc())
        .limit(100)  // batch processing
        .fetch();
}
```

| Retry | Wait Time | Purpose |
|-------|-----------|---------|
| 0 | 10 min | First query — allow time for confirm/webhook |
| 1 | 30 min | Retry — wait for transient failure recovery |
| 2 | 60 min | Final attempt — manual review needed after |

## Result

- confirm API failure → **webhook recovers**
- Webhook also fails → **scheduler queries PG directly for final correction**
- All 3 paths (confirm, webhook, scheduler) can run concurrently — **paymentKey + orderId UNIQUE constraint prevents duplicate records**
- Payment duplication and loss rate: **0%**

## Reflection

If I had thought of idempotency as just "duplicate prevention," a single Redis check would have been enough. But payment systems require **tracing failure scenarios backwards**.

"What if the PG approved it but our server died?" → The webhook recovers it.
"What if the webhook never arrives?" → The scheduler asks the PG directly.

Turning the answers to these questions into architecture is what the triple safety net is. Not simple exception handling, but **tracing failure scenarios backwards and designing structural safety nets** — that was the core of this payment system.
