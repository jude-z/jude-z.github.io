---
title: "Payment Idempotency — Eliminating Duplicate Charges with Redis + DB Dual Layer"
project: "trip-planner-ai"
lang: en
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Redis
  - Payment
date: 2026-02-25
excerpt: "How I solved duplicate payment issues caused by client retries using Redis setIfAbsent + DB Idempotency dual-layer design."
---

## Problem

When integrating Toss Payments, if a client **retries the same payment request** due to network timeouts, the PG gateway could approve the charge twice — a duplicate payment.

The obvious solution is Redis-based idempotency, but **if Redis goes down, the idempotency check itself becomes impossible**. Leaving Redis as a single point of failure means duplicate payments occur precisely when things go wrong — the worst-case scenario.

The question was simple:
> "How do we prevent duplicate payments even when Redis is dead?"

## Action

![Idempotency Architecture](/assets/images/trip-payment/idempotency-architecture.png)

### Redis setIfAbsent — First Layer (Speed)

The client's `Idempotency-Key` header is stored in Redis via `SETNX (setIfAbsent)`. If the key already exists, the request is immediately rejected as a duplicate.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public ApiResponse confirm(String idempotencyKey, PaymentRequest paymentRequest, Long id) {

    // 1st check: Redis idempotency (atomic operation)
    Boolean success = redisTemplate.opsForValue()
        .setIfAbsent(idempotencyKey, "true", Duration.of(24L, ChronoUnit.HOURS));

    if (success == null) throw new CommonException(Status.REDIS_SERVER_ERROR);
    if (!success) throw new CommonException(Status.ALREADY_PAYMENT_REQUEST);

    // 2nd check: DB idempotency (Redis failure fallback)
    Optional<Idempotency> optionalIdempotency =
        queryDslIdempotencyRepository.findByIdempotencyKey(idempotencyKey);
    if (optionalIdempotency.isPresent()) {
        throw new CommonException(Status.ALREADY_PAYMENT_REQUEST);
    }
    idempotencyRepository.save(Idempotency.of(idempotencyKey));

    // PG approval request
    boolean confirm = paymentClient.confirm(paymentRequest);
    if (confirm) {
        paymentFacade.processConfirm(paymentRequest, id);
    }
    return ApiStatusResponse.of(Status.SUCCESS);
}
```

The key is the **atomicity** of `setIfAbsent`. Even if 10 retry requests arrive simultaneously, Redis's single-threaded nature ensures only one request receives `true`.

### DB Idempotency Table — Second Layer (Durability)

When Redis is healthy, duplicates are caught at the first layer. But **when Redis is down**, the DB guarantees idempotency. The `Idempotency` table stores the same key with a UNIQUE constraint, blocking duplicates at the database level.

```
Redis (speed) + DB (durability) = Idempotency with no single point of failure
```

### Why Propagation.NOT_SUPPORTED?

`NOT_SUPPORTED` is applied to `confirm()` to **avoid holding a DB connection during external PG API calls**. Toss Payments API responses can take several seconds — keeping a DB transaction open during that time would exhaust the connection pool.

The actual DB writes are handled by `PaymentFacade` with `REQUIRES_NEW` in an independent transaction:

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

- Redis (speed) + DB (durability) dual layer ensures **idempotency with no SPOF**
- `setIfAbsent` atomic operation blocks concurrent requests
- `NOT_SUPPORTED` + `REQUIRES_NEW` transaction propagation **separates external API calls from DB writes**

## Reflection

If I had thought of idempotency as just "duplicate prevention," a single Redis check would have been enough. But payment systems require **tracing failure scenarios backwards**. The single question "What if Redis dies?" was the starting point for the DB dual-layer design.

The same applies to transaction propagation. Using `@Transactional` with defaults works, but failing to recognize that an external API call inside a transactional method holds a DB connection for seconds leads to connection pool exhaustion in production — a lesson best learned before it becomes an incident.
