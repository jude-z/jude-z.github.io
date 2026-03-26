---
title: "결제 테스트 4단계 — Redis + DB 이중 멱등성과 Reconciliation으로 완전한 안전망"
project: "trip-planner-ai"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Redis
  - Payment
  - Reconciliation
  - Scheduling
date: 2026-03-12
excerpt: "Redis + DB 이중 멱등성, 외부 PG 직접 조회 폴백, Reconciliation 스케줄러까지 — 어떤 장애 상황에서도 결제가 유실되지 않는 구조를 테스트로 검증한다."
---

## Problem

3단계까지의 테스트로 확인한 것:

| 단계 | 보호 장치 | 한계 |
|------|----------|------|
| V1 | 없음 | PG 100건 중복 승인 |
| V2 | Redis만 | Redis 장애 시 멱등성 뚫림 (SPOF) |
| V3 | Redis + Webhook | Webhook 미수신 시 복구 불가 |

각 단계에서 하나의 장애 시나리오를 해결하면 **다음 장애 시나리오가 드러나는** 패턴이다. 최종적으로 필요한 것은:

1. **Redis가 죽어도** 멱등성이 보장되어야 한다 → DB 이중 체크
2. **DB 저장이 실패해도** 복구 경로가 있어야 한다 → Webhook
3. **Webhook마저 실패해도** 최종 보정이 가능해야 한다 → Reconciliation
4. **PG confirm이 실패해도** 결제 상태를 확인할 수 있어야 한다 → 외부 직접 조회

> "어떤 단일 컴포넌트가 죽어도 결제가 유실되지 않으려면?"

## Action

### 시스템 아키텍처 — 4중 안전망

![V4 아키텍처](/assets/images/trip-payment/v4-full-safety.png)

### V4 코드: Redis + DB 이중 멱등성 + 외부 조회 폴백

```java
public ApiResponse confirmExternalCheck(String idempotencyKey,
                                         PaymentRequest paymentRequest, Long id) {
    // 1층: Redis 멱등성 (속도)
    if (!idempotencyRedisManager.tryAcquire(idempotencyKey)) {
        throw new CommonException(Status.ALREADY_PAYMENT_REQUEST);
    }

    // 2층: DB 멱등성 (영속성)
    Optional<Idempotency> existing =
        queryDslIdempotencyRepository.findByIdempotencyKey(idempotencyKey);
    if (existing.isPresent()) {
        throw new CommonException(Status.ALREADY_PAYMENT_REQUEST);
    }
    idempotencyRepository.save(Idempotency.of(idempotencyKey));

    // 3층: PG 승인 + 외부 조회 폴백
    boolean confirm = paymentClient.confirm(paymentRequest);
    if (!confirm) {
        log.warn("[EXTERNAL_CHECK] 외부 결제 서버 오류 - paymentKey로 결제 상태 직접 조회");
        boolean paymentCompleted =
            paymentClient.checkPaymentStatus(paymentRequest.getPaymentKey());
        if (paymentCompleted) {
            idempotencyRedisManager.complete(idempotencyKey);
            return ApiStatusResponse.of(Status.SUCCESS);
        }
        idempotencyRedisManager.fail(idempotencyKey);
        throw new CommonException(Status.PAYMENT_SERVER_ERROR);
    }
    idempotencyRedisManager.complete(idempotencyKey);
    return ApiStatusResponse.of(Status.SUCCESS);
}
```

V2와의 차이점:

| 구분 | V2 | V4 |
|------|----|----|
| Redis 멱등성 | O | O |
| DB 멱등성 | X | **O (Idempotency 테이블)** |
| Redis 장애 대비 | catch로 스킵 | **DB가 보호** |
| PG confirm 실패 | 그대로 실패 | **외부 직접 조회** |
| DB 저장 | 즉시 | **Reconciliation에 위임** |

### Idempotency 테이블 — DB 레벨 중복 차단

```java
@Entity
public class Idempotency {
    @Id @GeneratedValue
    private Long id;

    @Column(unique = true)  // UNIQUE 제약조건
    private String idempotencyKey;
}
```

`idempotencyKey`에 UNIQUE 제약조건이 걸려 있어, **DB 레벨에서 동일 키의 중복 삽입을 원천 차단**한다. Redis가 죽어도 DB가 멱등성을 보장한다.

### 외부 PG 직접 조회 — confirm 실패 시 폴백

```java
// PaymentClient
public boolean checkPaymentStatus(String paymentKey) {
    try {
        ResponseEntity<JsonNode> response = restTemplate.exchange(
            url.replace("/confirm", "/" + paymentKey),
            HttpMethod.GET,
            new HttpEntity<>(getHeaders()),
            JsonNode.class
        );
        return response.getStatusCode() == HttpStatus.OK;
    } catch (Exception e) {
        log.error("외부 결제 상태 조회 실패: {}", e.getMessage());
        return false;
    }
}
```

confirm API가 타임아웃이나 네트워크 오류로 실패해도, **PG에 직접 GET 요청을 보내 결제 상태를 확인**한다. PG가 200 OK를 반환하면 결제는 실제로 승인된 것이다.

### Reconciliation 스케줄러 — 최종 보정

```java
@Service
public class ReconciliationService {

    @Scheduled(cron = "10 */1 * * * *")  // 매 1분마다 실행
    @Transactional
    public void reconciliation() {
        LocalDateTime now = LocalDateTime.now();
        List<TempPayment> pendingPayments =
            queryDslTempPaymentRepository.fetchPendingPayments(now);

        List<Payment> savePayments = new ArrayList<>();
        List<String> deletePaymentKeys = new ArrayList<>();
        List<String> addRetryPaymentKeys = new ArrayList<>();

        for (TempPayment tempPayment : pendingPayments) {
            try {
                ResponseEntity<JsonNode> response = restTemplate.exchange(
                    url + tempPayment.getPaymentKey(),
                    HttpMethod.GET,
                    new HttpEntity<>(getHeaders()),
                    JsonNode.class
                );

                if (response.getStatusCode() == HttpStatus.OK) {
                    // PG에서 승인 확인 → Payment 생성
                    savePayments.add(PaymentFactory.from(...));
                    deletePaymentKeys.add(tempPayment.getPaymentKey());
                } else {
                    addRetryPaymentKeys.add(tempPayment.getPaymentKey());
                }
            } catch (Exception e) {
                addRetryPaymentKeys.add(tempPayment.getPaymentKey());
            }
        }

        // 배치 처리
        paymentRepository.saveAll(savePayments);
        tempPaymentRepository.deleteByPaymentKeyInBatch(deletePaymentKeys);
        tempPaymentRepository.addRetry(addRetryPaymentKeys);
    }
}
```

### 지수 백오프 재시도 조건

```java
public List<TempPayment> fetchPendingPayments(LocalDateTime now) {
    BooleanExpression retryCondition =
        tp.retry.eq(0).and(tp.createdTime.loe(now.minusMinutes(10)))
        .or(tp.retry.eq(1).and(tp.createdTime.loe(now.minusMinutes(30))))
        .or(tp.retry.eq(2).and(tp.createdTime.loe(now.minusMinutes(60))));

    return queryFactory.selectFrom(tp)
        .where(tp.status.eq(PaymentStatus.PENDING), retryCondition)
        .orderBy(tp.id.asc())
        .limit(100)
        .fetch();
}
```

| retry | 대기 시간 | 의미 |
|-------|----------|------|
| 0 | 10분 | confirm/Webhook이 처리할 시간 확보 |
| 1 | 30분 | 일시적 장애 복구 대기 |
| 2 | 60분 | 최종 시도, 이후 수동 확인 |

매 1분마다 스케줄러가 실행되지만, 실제 PG 조회는 retry 조건에 부합하는 건만 처리한다. PG에 불필요한 부하를 주지 않으면서도 **최대 60분 이내에 복구**가 완료된다.

## Result

### confirm 직후: Payment 0건

![V4 TempPayment](/assets/images/trip-payment/v4-temppayment.png)

`temp_payment`에 1건이 PENDING 상태로 남아 있다. V4는 confirm 시점에 `processConfirm()`을 호출하지 않으므로 Payment가 즉시 생성되지 않는다.

![V4 Payment 이전](/assets/images/trip-payment/v4%20이전%20payment.png)

`payment` 테이블은 0건. PG 승인은 완료됐지만 DB에는 아직 반영되지 않았다.

![V4 PG 서버](/assets/images/trip-payment/v4%20pg%20payment.png)

mock PG 서버에는 1건 저장. Redis + DB 이중 멱등성이 중복 요청을 완벽히 차단했다.

### Reconciliation 후: Payment 1건 복구

![V4 Reconciliation 후](/assets/images/trip-payment/v4%20reconcilation이후.png)

Reconciliation 스케줄러가 실행된 후, `payment` 테이블에 **1건이 COMPLETE 상태**로 생성됐다. 포인트도 10,000원이 정상 적립됐다.

| 시점 | Payment | PG mock | 멱등성 | 포인트 |
|------|---------|---------|--------|--------|
| confirm 직후 | 0건 | 1건 | Redis+DB 보호 | 0원 |
| **Reconciliation 후** | **1건 COMPLETE** | 1건 | 보장됨 | **10,000원** |

## 4단계 전체 비교

| | V1 | V2 | V3 | V4 |
|---|---|---|---|---|
| **멱등성** | 없음 | Redis | Redis | Redis + DB |
| **PG 중복 호출** | 100건 | 1건 | 1건 | 1건 |
| **Redis 장애 시** | - | 멱등성 뚫림 | 멱등성 뚫림 | **DB가 보호** |
| **DB 저장 실패 시** | 유실 | 유실 | Webhook 복구 | **Reconciliation 복구** |
| **Webhook 실패 시** | - | - | 복구 불가 | **Reconciliation 복구** |
| **PG confirm 실패 시** | 유실 | 유실 | 유실 | **외부 직접 조회** |
| **데이터 정합성** | PG 100 / DB 1 | 정상 | 정상 | **정상** |

V4는 **어떤 단일 컴포넌트가 장애를 일으켜도 결제가 유실되지 않는** 구조다.

## Reflection

이 시리즈의 본질은 "멱등성 구현"이 아니라 **장애 시나리오의 역추적**이었다.

- V1에서 "동시 요청이 들어오면?" → 중복 결제 발생 → Redis 멱등성 도입
- V2에서 "Redis가 죽으면?" → SPOF 노출 → DB 이중 구조 필요
- V3에서 "DB 저장이 실패하면?" → 결제 유실 → Webhook 복구 검증
- V4에서 "Webhook도 안 오면?" → Reconciliation 스케줄러

각 단계에서 "이것이 실패하면?"이라는 질문에 답하면서 안전망을 하나씩 쌓았다. 최종 구조는 설계부터 나온 것이 아니라, **실패 시나리오를 하나씩 증명하고 해결하는 과정**에서 만들어진 것이다.

결제 시스템에서 중요한 것은 "정상 경로"가 아니라 **"비정상 경로에서도 정합성을 유지하는 구조"**다. 테스트가 이를 증명하지 못하면, 운영 환경에서 장애가 증명해줄 것이다.

