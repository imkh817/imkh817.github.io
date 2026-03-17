---
title: "[3탄] SSE 알림, 왜 가끔 안 올까 — Redis Pub/Sub으로 멀티 인스턴스 브로드캐스트 구현"
series: "[블로그 프로젝트] 알림 구조 개선"
date: 2026-03-17 20:09:21 +0900
categories: [Redis 활용]
image:
  path: /assets/img/thumbnails/공부해야지.png
---

## 개요

{% include bookmark.html url="https://imkh817.github.io/posts/[1탄]-멀티인스턴스-환경을-고려한-구독-알림-구조-개선/" %}

2탄에서는 Consumer가 처리에 실패했을 때 메시지를 유실하지 않고 재처리하는 방법을 정리했다.
Redis Streams로 알림 DB 저장까지는 해결했지만, SSE 전송은 여전히 Spring Event에 의존하고 있어 멀티 인스턴스 환경에서 알림이 전달되지 않는 문제가 남아 있다.
1탄과 비슷한 문제지만, 이번엔 `Redis Streams` 대신 `Redis Pub/Sub`을 선택했다.

## 문제 상황
현재 구조에서 SSE 전송은 아래와 같이 동작한다.
```
Instance A (Redis Consumer 실행)                Instance B
    → 알림 DB 저장
    → Spring Event 발행
    → NotificationEventListener 실행
    → SSE 전송 시도 (userId=5)                 SSE 연결: userId=5 ← 여기 있음
    → 연결 없음 → 전송 실패
```
- Spring Event는 같은 JVM 안에서만 동작
- Redis Consumer가 Instance A에서 실행되면 SSE 전송도 Instance A 안에서만 시도
- SSE 연결이 Instance B에 있다면 알림은 전달 X

## Redis Streams가 아니라 Redis Pub/Sub을 선택한 이유
1탄에서 멀티 인스턴스 문제를 Redis Streams로 해결했는데, 이번엔 왜 다른 방식을 선택했을까?

Redis Streams의 Consumer Group은 메시지를 인스턴스 중 하나에게만 전달한다. 알림 DB 저장처럼 한 번만 처리되어야 하는 작업에는 적합하지만, SSE 전송에는 맞지 않는다.

**SSE 연결이 어느 인스턴스에 있는지 알 수 없기 때문에, 모든 인스턴스에 메시지를 전달하고 해당 연결을 가진 인스턴스가 처리해야 한다.** 그래서 `Redis Pub/Sub`을 사용해 브로드 캐스트 방식으로 구현하기로 결정했다.

| | Redis Streams | Redis Pub/Sub |
|---|---|---|
| 전달 방식 | 인스턴스 중 하나만 처리 | 모든 인스턴스에 브로드캐스트 |
| 메시지 저장 | 영속적으로 저장 | 휘발성 (저장 안됨) |
| 재처리 | 가능 | 불가능 |
| 적합한 상황 | 알림 DB 저장 (중복 방지) | SSE 전송 (브로드캐스트) |


또한 SSE 전송이 실패하더라도 알림은 이미 DB에 저장되어 있기 때문에 재처리는 필요 없다. 이후 클라이언트가 접속하면 읽지 않은 알림을 다시 조회할 수 있다.

## 구현
**발행 — NotificationPubSubPublisher**

알림 DB 저장이 완료된 후 `notification:sse` 채널로 메시지를 발행한다.
```java
@Component
@RequiredArgsConstructor
public class NotificationPubSubPublisher {

    static final String CHANNEL = "notification:sse";

    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;

    public void publish(NotificationSseEvent event) {
        try {
            String message = objectMapper.writeValueAsString(event);
            redisTemplate.convertAndSend(CHANNEL, message);
        } catch (JsonProcessingException e) {
            log.error("SSE 알림 직렬화 실패 - receiverId={}", event.receiverId(), e);
        }
    }
}
```

**구독 — NotificationPubSubListener**
채널에 메시지가 발행되면 모든 인스턴스의 리스너가 수신한다.
SSE 연결이 있는 인스턴스만 실제로 전송하고, 없으면 무시한다.
```java
@Component
@RequiredArgsConstructor
public class NotificationPubSubListener implements MessageListener {

    private final NotificationSseService notificationSseService;
    private final ObjectMapper objectMapper;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            NotificationSseEvent event = objectMapper.readValue(message.getBody(), NotificationSseEvent.class);
            notificationSseService.send(event.receiverId(), event.type(), event);
        } catch (Exception e) {
            log.error("SSE 알림 처리 실패", e);
        }
    }
}

public void send(Long memberId, NotificationType type, Object data) {
    SseEmitter emitter = sseEmitterRepository.get(memberId);

    if (emitter == null) {
        log.info("연결된 SSE 없음 - memberId={}", memberId);
        return;
    }

    try {
        emitter.send(SseEmitter.event()
                .name(type.name())
                .data(data));
    } catch (IOException e) {
        log.warn("SSE 전송 실패 - memberId={}", memberId, e);
        sseEmitterRepository.delete(memberId);
        emitter.complete();
    }
}

```

**설정 — NotificationPubSubConfig**
```java
@Configuration
@RequiredArgsConstructor
public class NotificationPubSubConfig {

    private final RedisConnectionFactory redisConnectionFactory;
    private final NotificationPubSubListener notificationPubSubListener;

    @Bean
    public RedisMessageListenerContainer notificationListenerContainer() {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(redisConnectionFactory);
        container.addMessageListener(
                notificationPubSubListener,
                new ChannelTopic(NotificationPubSubPublisher.CHANNEL)
        );
        return container;
    }
}
```

**트랜잭션 커밋 이후 발행 보장**
1탄과 동일하게 `@TransactionalEventListener(AFTER_COMMIT)`을 사용해 커밋 이후 Pub/Sub 발행을 보장했다.
```java
// SubscriptionNotificationProcessor
@Transactional
public void process(Long subscriberId, Long targetId, String messageId) {
    // 알림 DB 저장
    NotificationResponse notification = notificationCommandService.create(...);

    // Spring Event 발행
    eventPublisher.publishEvent(new NotificationSseEvent(...));
}

// NotificationEventListener
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(NotificationSseEvent event) {
    pubSubPublisher.publish(event);  // 커밋 이후 Pub/Sub 발행
}
```

### 전체 흐름
```plan text
Redis Consumer (알림 DB 저장 완료)
      │
      ▼  트랜잭션 커밋
Redis Pub/Sub 발행 (notification:sse 채널)
      │
      ├─ Instance A: SSE 연결 없음 → 무시
      └─ Instance B: SSE 연결 있음 → 전송 
```

## 마무리
Redis Pub/Sub을 도입해 SSE 알림을 모든 인스턴스에 브로드캐스트하는 구조로 바꿨다.

1탄부터 3탄까지 진행하면서 느낀 건, 기능이 동작하는 것과 멀티 인스턴스 환경에서도 잘 동작하는 건 다른 문제라는 점이었다.하나를 해결하면 또 다른 문제가 보이고, 생각보다 고려할 게 많았다.

그래도 직접 구현해보면서 Redis Streams와 Pub/Sub을 각각 어떤 상황에서 써야 하는지, 트랜잭션 경계를 어떻게 다뤄야 하는지 등을 배울 수 있어 뿌듯한 경험이였다.



