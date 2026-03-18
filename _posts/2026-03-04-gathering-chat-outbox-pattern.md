---
title: "Outbox Pattern으로 메시지 유실 0% 달성"
project: "gathering-server"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Kafka
  - Outbox Pattern
date: 2026-03-04
excerpt: "TransactionalEventListener로 DB-Kafka 정합성을 보장하고, 10초 주기 재발행으로 메시지 유실을 원천 차단한 설계."
---

## Problem

Send/Reply로 서비스를 분리하면서, 새로운 문제가 생겼다. **Kafka 발행이 실패하면 메시지가 유실**된다.

DB에 이벤트를 저장하고 Kafka에 발행하는 두 작업은 서로 다른 시스템이라 **원자적으로 처리할 수 없다**. DB 커밋은 성공했는데 Kafka 발행이 실패하면? 또는 Kafka 발행은 성공했는데 DB 커밋이 롤백되면?

> "DB와 Kafka를 어떻게 정합성 있게 묶을 수 있는가?"

## Action

### Outbox Pattern — DB 트랜잭션 안에 이벤트를 기록

Outbox 테이블에 이벤트를 저장하고, **트랜잭션 커밋 후** Kafka로 발행한다.

```java
@Entity
public class Outbox {
    @Id
    @Column(name = "outbox_id")
    private Long outboxId;

    @Enumerated(EnumType.STRING)
    private EventType eventType;

    private String payload;
    private LocalDateTime createdAt;
    private boolean processed;

    public static Outbox create(Event event, LocalDateTime localDateTime) {
        EventType eventType = event.getType();
        EventPayload payload = event.getPayload();
        Long eventId = payload.getEventId();
        String serializedPayload = DataSerializer.serialize(payload);

        return Outbox.builder()
                .eventType(eventType)
                .payload(serializedPayload)
                .createdAt(localDateTime)
                .outboxId(eventId)
                .build();
    }
}
```

### TransactionalEventListener — 이벤트 발행 시점 제어

핵심은 **BEFORE_COMMIT과 AFTER_COMMIT의 조합**이다.

```java
@Component
@RequiredArgsConstructor
public class EventListener {
    private final OutboxRepository outboxRepository;
    private final KafkaProducer kafkaProducer;

    // 1. 트랜잭션 커밋 전에 Outbox에 저장
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void createOutbox(Event<EventPayload> event) {
        Outbox outbox = Outbox.create(event, LocalDateTime.now());
        outboxRepository.save(outbox);
    }

    // 2. 트랜잭션 커밋 후에 Kafka 발행
    @Async("publishEventExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void publishEvent(Event<EventPayload> event) {
        kafkaProducer.publishEvent(event);
    }
}
```

**BEFORE_COMMIT**: Outbox 저장이 비즈니스 로직과 같은 트랜잭션에 포함된다. 트랜잭션이 롤백되면 Outbox도 같이 롤백되므로, **커밋되지 않은 이벤트가 발행될 일이 없다.**

**AFTER_COMMIT**: 트랜잭션이 확실히 커밋된 후에만 Kafka로 발행한다. DB에 저장되지 않은 이벤트가 Kafka에 먼저 도착하는 상황을 방지한다.

### 10초 주기 재발행 스케줄러

AFTER_COMMIT에서 Kafka 발행이 실패하면? Outbox에 `processed = false`로 남아 있으므로, **스케줄러가 주기적으로 미처리 이벤트를 재발행**한다.

```java
@Component
public class KafkaProducer {

    @Scheduled(fixedDelay = 10, initialDelay = 5, timeUnit = TimeUnit.SECONDS,
               scheduler = "publishPendingEventExecutor")
    public void publishPendingEvent() {
        List<Outbox> outboxes = outboxRepository
            .findAllByProcessedFalseAndCreatedAtBefore(
                PageRequest.of(0, 100, Sort.by("createdAt").descending()),
                LocalDateTime.now());

        outboxes.forEach(outbox -> {
            EventType eventType = outbox.getEventType();
            Class<? extends EventPayload> payloadClass = eventType.getPayloadClass();
            EventPayload payload = DataSerializer.deserialize(
                outbox.getPayload(), payloadClass);

            kafkaTemplate.send(
                eventType.getTopic(),
                String.valueOf(payload.getPartitionKey()),
                DataSerializer.serialize(payload));
        });
    }
}
```

100건씩 배치 처리하여 부하를 최소화하고, `createdAt` 역순으로 조회하여 최신 미처리 이벤트부터 처리한다.

## Result

- **BEFORE_COMMIT**: 이벤트 저장이 비즈니스 로직과 원자적으로 처리
- **AFTER_COMMIT**: DB 커밋 확인 후에만 Kafka 발행
- **스케줄러**: Kafka 장애 시에도 10초 주기로 미처리 이벤트 재발행
- 메시지 유실률 **0%** 달성

## Reflection

Outbox Pattern은 "DB와 메시지 브로커의 정합성"이라는 분산 시스템의 고전적 문제를 해결한다. `TransactionalEventListener`의 `BEFORE_COMMIT`과 `AFTER_COMMIT`을 조합하여 이벤트 발행 시점을 제어한 것이 설계의 핵심이었다.

단순히 Kafka에 보내기만 하면 되는 것이 아니라, **"보냈는데 안 갔으면 어떻게 하지?"**라는 질문에 대한 답이 Outbox 스케줄러였다.
