---
title: "결제 테스트 1단계 — 멱등성 없이 100개의 스레드 동시 결제, 무슨 일이 벌어지는가"
project: "trip-planner-ai"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Payment
  - Toss Payments
  - Concurrency
date: 2026-03-12
excerpt: "멱등성 보호 없이 100개의 동시 결제 요청을 보내면 PG에는 100건, 실제 DB에는 1건만 저장되는 데이터 불일치를 테스트로 증명한다."
---

## Problem

토스페이먼츠 연동 결제 시스템에서 **동시에 동일한 결제 요청이 여러 번 들어오면** 어떤 일이 벌어지는가?

클라이언트의 네트워크 타임아웃, 버튼 더블 클릭, 혹은 로드밸런서 재시도 등으로 동일한 결제 요청이 여러 번 서버에 도달할 수 있다. 멱등성 처리가 없다면 **모든 요청이 PG사에 승인 요청**으로 들어간다.

이 테스트의 목적은 단순하다:
> "보호 장치 없이 100개의 동시 결제 요청을 보내면, 정확히 어떤 데이터 불일치가 발생하는가?"

## Action

### 시스템 아키텍처 — 보호 없는 결제 흐름

![V1 시스템 아키텍처 — 멱등성 없음, 중복 결제 발생](/assets/images/trip-payment/v1-no-idempotency.png)

V1 confirm 메서드는 어떤 멱등성 체크도 없이, 요청이 들어오면 **즉시 PG에 승인을 요청**한다.

```java
public ApiResponse confirmNoIdempotency(PaymentRequest paymentRequest, Long id) {
    // 멱등성 체크 없음 — 모든 요청이 그대로 PG로 전달
    boolean confirm = paymentClient.confirm(paymentRequest);
    if (confirm) {
        paymentFacade.processConfirm(paymentRequest, id);
    }
    return ApiStatusResponse.of(Status.SUCCESS);
}
```

Redis 체크도 없고, DB 중복 확인도 없다. 100개의 스레드가 동시에 이 메서드를 호출하면, **100개가 모두 paymentClient.confirm()에 도달**한다.

### 테스트 코드 — 100개의 스레드 동시 요청

```java
@Test
@DisplayName("V1: 멱등성 없음 — 동시 100건 요청 시 중복 결제 발생")
void v1_noIdempotency_concurrentRequests() throws InterruptedException {
    int threadCount = 100;
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
    CountDownLatch latch = new CountDownLatch(threadCount);
    AtomicInteger successCount = new AtomicInteger(0);
    AtomicInteger failCount = new AtomicInteger(0);

    String body = """
            {
                "paymentKey": "paymentKey1",
                "orderId": "order1",
                "amount": 10000
            }
            """;

    for (int i = 0; i < threadCount; i++) {
        executorService.submit(() -> {
            try {
                // 동일한 paymentKey로 100개 동시 요청
                int status = sendConfirmRequest("/api/v1/confirm", body);
                if (status == 200) successCount.incrementAndGet();
                else failCount.incrementAndGet();
            } finally {
                latch.countDown();
            }
        });
    }
    latch.await();
}
```

100개의 스레드가 **동일한 paymentKey1**으로 동시에 `/api/v1/confirm`을 호출한다. CountDownLatch로 모든 스레드가 완료될 때까지 대기한다.

## Result

### 테스트 실행 결과

![V1 테스트 화면](/assets/images/trip-payment/v1%20테스트화면.png)

IntelliJ 테스트 콘솔에서 확인된 결과:
- **성공: 1건, 실패: 99건**
- 실패 원인: `404 Not Found Temp Payment` — 첫 번째 요청이 TempPayment를 삭제한 후, 나머지 99개의 요청이 TempPayment를 찾지 못해 실패

언뜻 보면 "1건만 성공했으니 문제 없는 것 아닌가?"라고 생각할 수 있다. 하지만 **PG서버 쪽 기록을 보면 상황이 다르다.**

### PG 서버 DB — 100건 저장

![V1 PG서버 DB](/assets/images/trip-payment/v1%20pg서버%20db저장내역.png)

mock PG 서버의 `payment` 테이블에는 **동일한 paymentKey1으로 100건이 저장**되어 있다. 모든 요청이 PG 승인까지 도달했다는 뜻이다. PG 서버 입장에서는 100번의 결제 승인이 발생한 것이다.

### 실제 DB — 1건 저장

![V1 실제 DB](/assets/images/trip-payment/v1%20실제%20db%20저장내역.png)

실제 서비스 DB의 `payment` 테이블에는 **1건만 COMPLETE 상태**로 저장되어 있다.

### 데이터 불일치의 본질

| 저장소 | 건수 | 상태 |
|--------|------|------|
| PG 서버 (mock.payment) | **100건** | 전부 승인됨 |
| 실제 서비스 DB (payment) | **1건** | COMPLETE |

PG에서는 100번 돈을 빼갔는데, 서비스에는 1건만 기록되어 있다. **99건의 결제가 유령 결제**가 된 것이다. 실제 운영 환경이었다면 유저의 돈은 100번 빠져나가고, 서비스에는 1건만 반영되는 치명적인 상황이다.

## Reflection

이 테스트가 증명하는 것은 단순하다: **멱등성 없는 결제 시스템은 동시 요청에 취약하다.**

흥미로운 점은 서버 응답만 보면 "1건 성공, 99건 실패"로 문제가 없어 보인다는 것이다. 하지만 실패 응답이 돌아오기 **전에** PG 승인은 이미 완료된 상태다. HTTP 응답 코드가 실패라고 해서 PG 승인이 취소되는 것이 아니다.

결제 시스템의 정합성은 **우리 서버의 응답이 아니라, PG 서버의 상태**를 기준으로 판단해야 한다. 이 간극을 메우는 것이 멱등성의 역할이다.

---

> PG에 100번 요청이 들어가는 것 자체를 막아야 한다. 첫 번째 요청만 통과시키고 나머지는 차단하려면?
> [2단계: Redis setIfAbsent로 동시 요청 차단 →](/projects/trip-payment-v2-redis-idempotency/)
