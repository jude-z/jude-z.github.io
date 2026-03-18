---
title: "TempPayment → Payment Transition Pattern and Transaction Propagation Strategy"
project: "trip-planner-ai"
lang: en
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - JPA
  - Transaction
date: 2026-03-11
excerpt: "Separating external PG calls from DB writes with transaction propagation, and structurally detecting payment loss via TempPayment → Payment transition."
---

## Problem

Two structural issues existed in the payment flow:

**1. DB Connection Held During External API Calls**

With the default `@Transactional` (REQUIRED), wrapping the confirm method means the DB connection is held during the entire Toss Payments API call (several seconds). Under concurrent payment requests, **the connection pool gets exhausted**, bringing down the entire service.

**2. No Way to Detect Payment Loss After PG Approval**

In a design where Payment is created directly, if the PG approves but the DB write fails, **there's no way to detect the lost payment**. The user's money is charged, but the service has no record.

## Action

![Transaction Propagation Strategy](/assets/images/trip-payment/transaction-propagation.png)

### NOT_SUPPORTED — External API Call Zone

`Propagation.NOT_SUPPORTED` is applied to `PaymentService.confirm()`. This propagation attribute **suspends the existing transaction** and executes the method without one.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public ApiResponse confirm(String idempotencyKey, PaymentRequest paymentRequest, Long id) {
    // Redis idempotency check — no transaction needed
    Boolean success = redisTemplate.opsForValue()
        .setIfAbsent(idempotencyKey, "true", Duration.of(24L, ChronoUnit.HOURS));
    ...

    // DB idempotency check — no transaction needed (save is separate)
    idempotencyRepository.save(Idempotency.of(idempotencyKey));

    // External PG API call — no DB connection held
    boolean confirm = paymentClient.confirm(paymentRequest);

    if (confirm) {
        // DB writes delegated to separate transaction
        paymentFacade.processConfirm(paymentRequest, id);
    }
    return ApiStatusResponse.of(Status.SUCCESS);
}
```

Even if the external PG API call takes 3 seconds, no DB connection is held during that time.

### REQUIRES_NEW — DB Write Zone

`Propagation.REQUIRES_NEW` is applied to `PaymentFacade`. It starts a **completely independent new transaction** regardless of the parent.

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

        // 1. Retrieve TempPayment (PENDING status)
        TempPayment tempPayment = tempPaymentRepository
            .findByPaymentKeyAndOrderIdAndAmountAndStatus(
                paymentKey, orderId, amount, PaymentStatus.PENDING)
            .orElseThrow(() -> new CommonException(Status.NOT_FOUND_PAYMENT));

        // 2. Create Payment (COMPLETE status)
        Payment payment = PaymentFactory.from(paymentRequest, member);
        paymentRepository.save(payment);

        // 3. Add points
        Point point = pointRepository.findByMember(member)
            .orElseThrow(() -> new CommonException(Status.NOT_FOUND_POINT));
        point.addAmount(amount);

        // 4. Delete TempPayment
        tempPaymentRepository.delete(tempPayment);
    }
}
```

If an exception occurs within this method, Payment save, point addition, and TempPayment deletion are **all rolled back**. Atomicity is guaranteed — no intermediate states exist.

### TempPayment → Payment Transition Pattern

The core of this design is **separating payment intent from payment completion**.

| Entity | Role | Status |
|--------|------|--------|
| TempPayment | Records payment intent | PENDING |
| Payment | Records payment completion | COMPLETE |

```
Client payment request
  → TempPayment created (PENDING)
    → PG approval + idempotency check
      → Payment created (COMPLETE)
        → TempPayment deleted
```

**Why is TempPayment Necessary?**

In a design where Payment is created directly, if the DB write fails after PG approval, the payment loss is **undetectable**. But with TempPayment remaining:

- The Reconciliation scheduler discovers PENDING TempPayments
- Queries the PG directly to verify approval status
- If approved, creates Payment to recover

TempPayment serves as a **signal to the system that "this payment hasn't been completed yet."**

### Transaction Propagation Summary

| Component | Propagation | Reason |
|-----------|-------------|--------|
| PaymentService.confirm() | NOT_SUPPORTED | Prevent DB connection holding during PG calls |
| PaymentService.cancel() | NOT_SUPPORTED | Same reason |
| PaymentFacade | REQUIRES_NEW | Guarantee DB write atomicity, independent rollback |

## Result

- External PG API calls and DB writes **completely separated at the transaction level**
- `NOT_SUPPORTED` prevents connection pool exhaustion during PG calls
- `REQUIRES_NEW` guarantees **atomicity** of Payment save, point addition, and TempPayment deletion
- TempPayment → Payment transition pattern enables **structural detection of payment loss**

## Reflection

`@Transactional` is convenient, but defaults alone cannot safely handle a payment flow that includes external API calls. Transaction boundaries must be designed intentionally.

The TempPayment → Payment transition pattern is not just a status change — it's a **structural mechanism that lets the system recognize "payments that haven't been completed yet."** Without this pattern, the Reconciliation scheduler in the triple safety net could not function.

Ultimately, the transaction propagation strategy (NOT_SUPPORTED + REQUIRES_NEW) and the TempPayment → Payment transition pattern are not independent techniques — they are **designs that interlock to serve a single goal: payment safety**.
