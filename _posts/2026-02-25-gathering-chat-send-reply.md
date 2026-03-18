---
title: "모놀리식 채팅의 한계와 Send/Reply 분리 설계"
project: "gathering-server"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - Kafka
  - WebSocket
date: 2026-02-25
excerpt: "DB 저장 지연이 실시간 응답을 마비시키는 모놀리식 구조를 Send/Reply 마이크로서비스로 분리한 과정."
---

## Problem

모놀리식 chat-server에서 WebSocket 연결, 메시지 저장, 읽음 처리를 **한 프로세스가 담당**했다.

DB 저장 지연이 WebSocket 실시간 응답에 직접 영향을 주고, **하나의 장애가 전체 채팅을 마비시키는 구조**였다. 메시지 저장 실패 시 복구 경로가 없어 유실이 발생했고, WebSocket 커넥션 증가에 따른 DB 부하를 독립적으로 스케일링할 수 없었다.

> "DB가 느려지면 채팅 응답도 느려진다" — 이 문제가 서비스 분리의 출발점이었다.

## Action

![Send/Reply 분리 아키텍처](/assets/images/gathering/chat-send-reply-architecture.png)

실시간 응답과 영속화를 분리하기 위해 **두 개의 서비스로 관심사를 나눴다**.

### chat-send-server — WebSocket 즉시 브로드캐스트

WebSocket 메시지 수신 즉시 이벤트를 생성하고, Kafka로 비동기 발행한다. **DB 저장 완료를 기다리지 않는다.**

```java
@Controller
@RequiredArgsConstructor
public class SendController {

    private final KafkaProducer kafkaProducer;
    private final IdGenerator idGenerator;
    private final ChatService chatService;

    @MessageMapping("/{chatRoomId}")
    public void sendMessage(@DestinationVariable Long chatRoomId,
                            @Payload ChatMessagePayload chatMessagePayload) {
        Event event = createChatMessageEvent(chatRoomId, chatMessagePayload);
        chatService.handle(event);
    }

    public Event createChatMessageEvent(Long chatRoomId, ChatMessagePayload payload) {
        Long chatSendId = idGenerator.nextIdForReply();
        payload.setPartitionKey(chatRoomId);
        payload.setEventId(chatSendId);
        return Event.builder()
                .payload(payload)
                .type(EventType.CHAT_MESSAGE)
                .build();
    }
}
```

`chatRoomId`를 파티션 키로 설정하여 **같은 채팅방의 메시지는 같은 파티션**에 들어간다. 이로써 방 단위 메시지 순서가 보장된다.

### Kafka Consumer — 실시간 브로드캐스트

Kafka에서 메시지를 소비하여 WebSocket 구독자에게 즉시 전달한다.

```java
@Component
public class ChatConsumer {

    private final SimpMessageSendingOperations messageSendingOperations;

    @KafkaListener(topics = EventType.Topic.CHAT_MESSAGE,
        groupId = "#{T(java.util.UUID).randomUUID().toString()}")
    public void chatMessageListen(String message, Acknowledgment ack) {
        Event<ChatMessagePayload> event = Event.fromJson(message);
        ChatMessagePayload payload = event.getPayload();
        Long chatRoomId = payload.getChatRoomId();

        messageSendingOperations.convertAndSend("/chatRoom/" + chatRoomId, payload);
        ack.acknowledge();
    }
}
```

`groupId`를 UUID로 생성하는 이유는, **모든 chat-server 인스턴스가 동일한 메시지를 수신**해야 하기 때문이다. 각 인스턴스에 연결된 WebSocket 클라이언트가 다르므로, 브로드캐스트를 위해 모든 인스턴스가 메시지를 받아야 한다.

### chat-reply-server — Kafka 컨슈머 기반 영속화

Kafka에서 메시지를 소비하여 ChatMessage 저장 + 전체 참여자 ReadStatus를 일괄 생성한다.

```java
@Service
public class ChatService {

    public void handle(Event<ChatMessagePayload> event) {
        ChatMessagePayload payload = event.getPayload();
        if (!validateEvent(payload)) return;  // Redis 멱등성 검증

        ChatParticipant sender = findSender(payload);
        ChatMessage chatMessage = saveChatMessage(sender);
        saveReadStatuses(chatMessage, sender.getChatRoom());
    }

    private boolean validateEvent(ChatMessagePayload payload) {
        String chatKey = generateKey(payload);
        return StringUtils.hasText(redisTemplate.opsForValue().get(chatKey));
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

Redis 멱등성 검증(`validateEvent`)으로 **Kafka 재전송에 의한 중복 처리를 방지**한다.

## Result

- 실시간 응답과 DB 저장을 **완전히 분리** → 응답 지연 제거
- 각 서비스를 **독립 스케일링** 가능
- DB 저장 실패가 실시간 응답에 영향을 주지 않음

## Reflection

모놀리식에서는 "한 곳에서 다 처리하니까 간단하다"고 생각했지만, **장애 전파 범위를 고려하면 관심사 분리가 필수**였다. "DB가 느려지면 채팅 응답도 느려진다"는 문제는 단순한 성능 이슈가 아니라, 아키텍처의 구조적 한계였다.
