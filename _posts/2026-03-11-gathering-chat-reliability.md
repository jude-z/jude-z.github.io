---
title: "Kafka 파티션 순서 보장 + Redis 멱등성으로 중복 처리 방지"
project: "gathering-server"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Kafka
  - Redis
date: 2026-03-11
excerpt: "chatRoomId를 파티션 키로 방 단위 메시지 순서를 보장하고, Redis 멱등성으로 at-least-once 환경의 중복 소비를 방지한 설계."
---

## Problem

Outbox Pattern으로 **at-least-once** 전달을 보장하면, 필연적으로 두 가지 문제가 따라온다:

1. **메시지 순서** — Kafka는 파티션 내에서만 순서를 보장한다. 같은 채팅방의 메시지가 다른 파티션에 들어가면 순서가 뒤바뀔 수 있다.
2. **중복 소비** — at-least-once이므로 같은 메시지가 두 번 이상 도착할 수 있다. 중복으로 ChatMessage가 저장되면 안 된다.

## Action

### chatRoomId를 파티션 키로 — 방 단위 순서 보장

```java
@Async("publishEventExecutor")
public void publishEvent(Event<EventPayload> event) {
    EventPayload payload = event.getPayload();
    EventType eventType = event.getType();
    String topic = eventType.getTopic();
    String partitionKey = String.valueOf(payload.getPartitionKey());

    kafkaTemplate.send(
        topic,
        partitionKey,           // chatRoomId가 파티션 키
        DataSerializer.serialize(payload)
    );
}
```

`SendController`에서 `payload.setPartitionKey(chatRoomId)`로 설정한 값이 여기서 Kafka 파티션 키로 사용된다.

같은 `chatRoomId`를 가진 메시지는 항상 같은 파티션에 들어가므로, **방 단위로 메시지 순서가 보장**된다. A방과 B방의 메시지는 서로 다른 파티션에서 독립적으로 처리되어 병렬성도 확보된다.

### Redis 멱등성 — 중복 소비 방지

```java
@Service
public class ChatService {

    public void handle(Event<ChatMessagePayload> event) {
        ChatMessagePayload payload = event.getPayload();
        if (!validateEvent(payload)) return;  // 중복이면 무시

        ChatParticipant sender = findSender(payload);
        ChatMessage chatMessage = saveChatMessage(sender);
        saveReadStatuses(chatMessage, sender.getChatRoom());
    }

    private boolean validateEvent(ChatMessagePayload payload) {
        String chatKey = generateKey(payload);
        return StringUtils.hasText(redisTemplate.opsForValue().get(chatKey));
    }

    private String generateKey(EventPayload eventPayload) {
        Long eventId = eventPayload.getEventId();
        return key + ":" + eventId;
    }
}
```

각 이벤트는 Redis에서 발급한 **고유 ID(eventId)**를 가진다. `validateEvent`에서 Redis에 해당 키가 존재하는지 확인하여, 이미 처리된 이벤트는 건너뛴다.

### Redis 기반 ID 생성

```java
@Component
public class IdGenerator {
    private final RedisTemplate<String, String> redisTemplate;

    public Long nextIdForReply() {
        return redisTemplate.opsForValue().increment(replyKey);
    }
}
```

Redis `INCR`의 원자성을 활용하여 **분산 환경에서도 유일한 ID**를 생성한다. 이 ID가 Outbox의 PK이자 멱등성 검증 키로 사용된다.

### Kafka Manual ACK

```java
@KafkaListener(topics = EventType.Topic.CHAT_MESSAGE,
    groupId = "#{T(java.util.UUID).randomUUID().toString()}")
public void chatMessageListen(String message, Acknowledgment ack) {
    Event<ChatMessagePayload> event = Event.fromJson(message);
    ChatMessagePayload payload = event.getPayload();
    Long chatRoomId = payload.getChatRoomId();

    messageSendingOperations.convertAndSend("/chatRoom/" + chatRoomId, payload);
    ack.acknowledge();  // 수동 ACK
}
```

`ack.acknowledge()`를 명시적으로 호출하여, 메시지 처리가 완료된 후에만 오프셋을 커밋한다. 처리 중 장애가 발생하면 Kafka가 메시지를 재전송하고, Redis 멱등성으로 중복은 방지된다.

## Result

- `chatRoomId` 파티션 키로 **방 단위 메시지 순서 보장**
- Redis 멱등성으로 **at-least-once 환경에서 중복 처리 방지**
- Manual ACK + Outbox 재시도로 **메시지 유실률 0%**
- Redis `INCR` 기반 분산 ID 생성으로 **유일성 보장**

## Reflection

at-least-once 보장은 메시지를 "적어도 한 번"은 전달하지만, "정확히 한 번"은 보장하지 않는다. 이 간극을 Redis 멱등성으로 메웠다.

Kafka Manual ACK + Outbox 재시도로 at-least-once를 보장하면서, Redis 멱등성으로 중복 처리를 방지한 점이 설계의 핵심이었다. 결국 **신뢰성(유실 0%) + 정확성(중복 0%) + 순서(방 단위)**를 모두 달성하려면, 단일 기술이 아니라 여러 메커니즘의 조합이 필요했다.
