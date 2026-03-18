---
title: "Monolithic Chat Limitations and Send/Reply Separation Design"
project: "gathering-server"
lang: en
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Kafka
  - WebSocket
date: 2026-02-25
excerpt: "How I split a monolithic chat server where DB latency paralyzed real-time responses into Send/Reply microservices."
---

## Problem

In the monolithic chat-server, WebSocket connections, message persistence, and read status management were **all handled by a single process**.

DB write latency directly affected WebSocket real-time responses, and **a single failure could paralyze the entire chat system**. There was no recovery path for failed message saves, and DB load from increasing WebSocket connections couldn't be scaled independently.

> "When the DB slows down, chat responses slow down too" — this was the starting point for service separation.

## Action

![Send/Reply Architecture](/assets/images/gathering/chat-send-reply-architecture.png)

To separate real-time responses from persistence, I **split concerns into two services**.

### chat-send-server — Immediate WebSocket Broadcast

Upon receiving a WebSocket message, an event is created and published to Kafka asynchronously. **It doesn't wait for DB writes to complete.**

```java
@Controller
@RequiredArgsConstructor
public class SendController {

    @MessageMapping("/{chatRoomId}")
    public void sendMessage(@DestinationVariable Long chatRoomId,
                            @Payload ChatMessagePayload chatMessagePayload) {
        Event event = createChatMessageEvent(chatRoomId, chatMessagePayload);
        chatService.handle(event);
    }

    public Event createChatMessageEvent(Long chatRoomId, ChatMessagePayload payload) {
        Long chatSendId = idGenerator.nextIdForReply();
        payload.setPartitionKey(chatRoomId);  // partition key = chatRoomId
        payload.setEventId(chatSendId);
        return Event.builder()
                .payload(payload)
                .type(EventType.CHAT_MESSAGE)
                .build();
    }
}
```

Setting `chatRoomId` as the partition key ensures **messages from the same chat room go to the same partition**, guaranteeing per-room message ordering.

### chat-reply-server — Kafka Consumer-Based Persistence

Consumes messages from Kafka to save ChatMessage + bulk-create ReadStatus for all participants.

```java
@Service
public class ChatService {

    public void handle(Event<ChatMessagePayload> event) {
        ChatMessagePayload payload = event.getPayload();
        if (!validateEvent(payload)) return;  // Redis idempotency check

        ChatParticipant sender = findSender(payload);
        ChatMessage chatMessage = saveChatMessage(sender);
        saveReadStatuses(chatMessage, sender.getChatRoom());
    }

    private void saveReadStatuses(ChatMessage chatMessage, ChatRoom chatRoom) {
        List<ChatParticipant> participants =
            chatParticipantRepository.findAllByChatRoom(chatRoom);
        List<ReadStatus> readStatuses = participants.stream()
            .map(p -> ReadStatusMapper.toReadStatus(p, chatMessage))
            .toList();
        readStatusRepository.saveAll(readStatuses);
    }
}
```

## Result

- Real-time responses and DB writes **completely separated** — response latency eliminated
- Each service can be **scaled independently**
- DB write failures don't affect real-time responses

## Reflection

In the monolithic design, "handling everything in one place seems simple" — but **considering failure propagation scope, separation of concerns was essential**. The problem "DB slowdown causes chat slowdown" wasn't just a performance issue; it was a structural limitation of the architecture.
