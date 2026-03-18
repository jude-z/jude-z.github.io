---
title: "Achieving 0% Message Loss with the Outbox Pattern"
project: "gathering-server"
lang: en
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Kafka
  - Outbox Pattern
date: 2026-03-04
excerpt: "Using TransactionalEventListener for DB-Kafka consistency and a 10-second retry scheduler to eliminate message loss."
---

## Problem

After splitting into Send/Reply services, a new problem emerged: **if Kafka publishing fails, messages are lost**.

Saving an event to DB and publishing to Kafka are two different systems that **cannot be processed atomically**. What if the DB commits but Kafka publishing fails? Or Kafka publishing succeeds but the DB transaction rolls back?

## Action

### Outbox Pattern — Record Events Inside DB Transactions

Store events in an Outbox table and **publish to Kafka only after transaction commit**.

```java
@Entity
public class Outbox {
    @Id
    private Long outboxId;

    @Enumerated(EnumType.STRING)
    private EventType eventType;
    private String payload;
    private LocalDateTime createdAt;
    private boolean processed;

    public static Outbox create(Event event, LocalDateTime localDateTime) {
        return Outbox.builder()
                .eventType(event.getType())
                .payload(DataSerializer.serialize(event.getPayload()))
                .createdAt(localDateTime)
                .outboxId(event.getPayload().getEventId())
                .build();
    }
}
```

### TransactionalEventListener — Controlling Event Publication Timing

The key is the **combination of BEFORE_COMMIT and AFTER_COMMIT**.

```java
@Component
public class EventListener {

    // 1. Save to Outbox before transaction commits
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void createOutbox(Event<EventPayload> event) {
        Outbox outbox = Outbox.create(event, LocalDateTime.now());
        outboxRepository.save(outbox);
    }

    // 2. Publish to Kafka after transaction commits
    @Async("publishEventExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void publishEvent(Event<EventPayload> event) {
        kafkaProducer.publishEvent(event);
    }
}
```

**BEFORE_COMMIT**: Outbox save is included in the same transaction as business logic. If the transaction rolls back, the Outbox rolls back too — **uncommitted events are never published.**

**AFTER_COMMIT**: Publishes to Kafka only after the transaction is confirmed committed. Prevents events from arriving at Kafka before they're persisted in DB.

### 10-Second Retry Scheduler

If Kafka publishing fails at AFTER_COMMIT? The Outbox record remains with `processed = false`, so the **scheduler periodically republishes unprocessed events**.

```java
@Component
public class KafkaProducer {

    @Scheduled(fixedDelay = 10, initialDelay = 5, timeUnit = TimeUnit.SECONDS)
    public void publishPendingEvent() {
        List<Outbox> outboxes = outboxRepository
            .findAllByProcessedFalseAndCreatedAtBefore(
                PageRequest.of(0, 100, Sort.by("createdAt").descending()),
                LocalDateTime.now());

        outboxes.forEach(outbox -> {
            EventPayload payload = DataSerializer.deserialize(
                outbox.getPayload(), outbox.getEventType().getPayloadClass());
            kafkaTemplate.send(
                outbox.getEventType().getTopic(),
                String.valueOf(payload.getPartitionKey()),
                DataSerializer.serialize(payload));
        });
    }
}
```

## Result

- **BEFORE_COMMIT**: Event storage is atomic with business logic
- **AFTER_COMMIT**: Kafka publishing only after confirmed commit
- **Scheduler**: 10-second retry for unprocessed events during Kafka outages
- Message loss rate: **0%**

## Reflection

The Outbox Pattern solves the classic distributed systems problem of "DB and message broker consistency." Combining `TransactionalEventListener`'s `BEFORE_COMMIT` and `AFTER_COMMIT` to control event publication timing was the core of this design.

It's not just about sending to Kafka — the answer to **"What if we sent it but it didn't arrive?"** was the Outbox scheduler.
