# Week9 — WebSocket 실시간 알림 구조 정리

## 9주차

이번 주제는 Monitory 프로젝트에서 사용한 WebSocket 실시간 알림 구조다.

Monitory는 센서 데이터가 계속 들어오는 실시간 모니터링 시스템이다. 사용자가 새로고침하거나 버튼을 누르지 않아도, 위험 상황이 발생하면 화면에 바로 알림이 떠야 한다. 이때 사용한 기술이 WebSocket이다.

전체 흐름은 다음과 같다.

```text
MQTT
→ Kafka
→ Spring Boot Kafka Consumer
→ SensorEventProcessor
→ AbnormalLog 저장
→ AlarmEventService
→ WebSocketSender
→ /topic/alarm
→ 프론트 화면 실시간 알림
```

한 문장으로 정리하면, Monitory의 WebSocket은 Kafka에서 처리된 이상 감지 결과를 프론트 화면에 실시간으로 전달하는 통로다.

---

# 1. WebSocket을 왜 사용했는가

일반적인 HTTP 요청은 클라이언트가 먼저 요청해야 서버가 응답한다.

예를 들어 사용자가 버튼을 누르거나, 화면을 새로고침하거나, 일정 시간마다 API를 호출해야 최신 데이터를 받을 수 있다.

```text
클라이언트: 새로운 알림 있어요?
서버: 네, 있어요.

클라이언트: 새로운 알림 있어요?
서버: 아니요, 없어요.
```

이 방식은 실시간 알림에는 적합하지 않다.

공장 모니터링 시스템에서는 온도, 습도, 진동, 작업자 생체 정보 같은 데이터가 계속 들어온다. 위험 상황이 발생했는데 사용자가 새로고침해야만 알림을 볼 수 있다면 실시간 대응이 어렵다.

그래서 서버가 먼저 클라이언트에게 메시지를 보낼 수 있는 WebSocket이 필요했다.

```text
서버: 위험 상황 발생했습니다. 바로 화면에 표시하세요.
클라이언트: 알림을 받았습니다.
```

즉, WebSocket은 서버가 프론트에게 실시간으로 메시지를 밀어주는 구조다.

---

# 2. HTTP Polling과 WebSocket 차이

HTTP Polling은 클라이언트가 일정 시간마다 서버에 계속 물어보는 방식이다.

```text
프론트 → 서버: 알림 있어?
프론트 → 서버: 알림 있어?
프론트 → 서버: 알림 있어?
```

이 방식은 구현이 쉽지만, 알림이 없어도 계속 요청을 보내기 때문에 불필요한 트래픽이 생긴다. 또 요청 주기가 5초라면 위험 상황이 발생해도 최대 5초 늦게 알림을 받을 수 있다.

WebSocket은 처음 한 번 연결을 맺은 뒤, 연결을 유지한다.

```text
프론트 ↔ 서버: 연결 유지
서버 → 프론트: 위험 알림 발생 시 바로 전송
```

그래서 실시간성이 필요한 알림, 채팅, 모니터링 화면에 적합하다.

Monitory에서는 공장 센서 데이터가 Kafka로 들어오고, 백엔드가 이를 처리한 뒤 WebSocket으로 화면에 바로 전달했다.

---

# 3. Monitory에서 WebSocket이 들어간 위치

Monitory의 실시간 알림 흐름은 아래와 같다.

```text
센서 시뮬레이터
→ MQTT
→ AWS IoT Core
→ Kinesis / Flink / Kafka
→ Monitory BE
→ WebSocket
→ Monitory FE
```

백엔드 기준으로 보면 더 단순하게 볼 수 있다.

```text
Kafka Consumer
→ SensorEventProcessor
→ 위험도 판단
→ AbnormalLog 저장
→ AlarmEventService
→ WebSocketSender
→ 프론트 구독 토픽으로 전송
```

여기서 WebSocketSender는 실제로 프론트에게 메시지를 보내는 역할을 한다.

주요 전송 토픽은 다음처럼 나눌 수 있다.

```text
/topic/alarm
→ 위험 알림 전송

/topic/zone
→ 공간별 위험도 변경 전송

/topic/unread-count
→ 읽지 않은 알림 개수 전송

/topic/control-status
→ 자동 제어 상태 전송
```

토픽을 나눈 이유는 화면에서 필요한 데이터 종류가 다르기 때문이다.

예를 들어 위험 알림 목록은 `/topic/alarm`을 구독하고, 공간별 위험도 히트맵은 `/topic/zone`을 구독할 수 있다. 이렇게 나누면 프론트는 필요한 주제만 골라서 받을 수 있다.

---

# 4. STOMP란 무엇인가

Monitory는 WebSocket 위에서 STOMP 방식을 사용한다.

WebSocket은 통신 연결 방식에 가깝다. 쉽게 말하면 서버와 클라이언트 사이에 길을 뚫어주는 기술이다.

그런데 길만 있다고 해서 메시지를 어떤 형식으로 주고받을지까지 정해지는 것은 아니다.

이때 사용하는 메시지 규칙이 STOMP다.

```text
WebSocket
→ 서버와 클라이언트 사이에 실시간 연결을 만든다.

STOMP
→ 그 연결 위에서 메시지를 어떤 주소로 보내고 받을지 정한다.
```

STOMP를 사용하면 `/topic/alarm` 같은 주소를 기준으로 메시지를 구독하고 발행할 수 있다.

프론트는 이렇게 특정 토픽을 구독한다.

```javascript
stompClient.subscribe("/topic/alarm", function(message) {
    console.log("위험 알림 수신:", message.body);
});
```

백엔드는 해당 토픽으로 메시지를 보낸다.

```java
messagingTemplate.convertAndSend("/topic/alarm", alarmResponse);
```

그러면 `/topic/alarm`을 구독 중인 모든 클라이언트가 같은 알림을 받는다.

---

# 5. WebSocketConfig의 역할

WebSocketConfig는 WebSocket 연결 주소와 메시지 브로커 설정을 담당한다.

설명용 예시는 다음과 같다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }
}
```

하나씩 보면 다음과 같다.

```java
@Configuration
```

이 클래스가 Spring 설정 클래스라는 뜻이다.

```java
@EnableWebSocketMessageBroker
```

Spring에서 WebSocket 메시지 브로커 기능을 켠다는 뜻이다.

```java
registry.addEndpoint("/ws")
```

프론트가 WebSocket 연결을 시작할 주소를 정한다.

즉, 프론트는 보통 다음 주소로 연결한다.

```text
http://localhost:8080/ws
```

```java
.setAllowedOriginPatterns("*")
```

프론트 서버와 백엔드 서버의 주소가 다를 수 있기 때문에 CORS 허용을 설정한다.

```java
.withSockJS()
```

WebSocket을 지원하지 않는 환경에서도 대체 방식으로 연결할 수 있게 도와준다.

```java
registry.enableSimpleBroker("/topic")
```

`/topic`으로 시작하는 주소를 구독용 메시지 브로커 대상으로 사용한다는 뜻이다.

```java
registry.setApplicationDestinationPrefixes("/app")
```

클라이언트가 서버로 메시지를 보낼 때 `/app`으로 시작하는 주소를 사용하게 한다.

Monitory에서는 주로 서버가 프론트로 알림을 보내는 구조였기 때문에 `/topic/alarm`, `/topic/zone` 같은 구독 토픽이 중요했다.

---

# 6. WebSocketSender의 역할

WebSocketSender는 실제 메시지를 프론트로 전송하는 클래스다.

설명용 예시는 다음과 같다.

```java
@Component
@RequiredArgsConstructor
public class WebSocketSender {

    private final SimpMessagingTemplate messagingTemplate;

    @Async
    public void sendDangerAlarm(AlarmEventResponse response) {
        messagingTemplate.convertAndSend("/topic/alarm", response);
    }

    @Async
    public void sendDangerLevel(ZoneDangerResponse response) {
        messagingTemplate.convertAndSend("/topic/zone", response);
    }

    @Async
    public void sendUnreadCount(Long unreadCount) {
        messagingTemplate.convertAndSend("/topic/unread-count", unreadCount);
    }

    @Async
    public void sendControlStatus(ControlStatusResponse response) {
        messagingTemplate.convertAndSend("/topic/control-status", response);
    }
}
```

핵심은 `SimpMessagingTemplate`이다.

```java
private final SimpMessagingTemplate messagingTemplate;
```

`SimpMessagingTemplate`은 Spring에서 STOMP 메시지를 특정 토픽으로 보내는 도구다.

```java
messagingTemplate.convertAndSend("/topic/alarm", response);
```

이 코드는 `/topic/alarm`을 구독 중인 프론트에게 `response` 객체를 JSON 형태로 보내는 역할을 한다.

---

# 7. 왜 @Async를 붙였는가

처음 구조에서는 Kafka Consumer가 메시지를 처리하면서 WebSocket 전송까지 같은 흐름에서 처리할 수 있었다.

이 구조는 문제가 있다.

Kafka Consumer는 계속 들어오는 메시지를 빠르게 처리해야 한다. 그런데 WebSocket 전송 I/O까지 Consumer 스레드가 직접 맡으면, 전송이 느려질 때 다음 Kafka 메시지 처리가 밀릴 수 있다.

즉, 하나의 스레드가 너무 많은 일을 하게 된다.

```text
Kafka Consumer 스레드
→ 메시지 수신
→ 위험도 계산
→ DB 저장
→ WebSocket 전송
→ 다음 메시지 처리
```

이 구조에서는 WebSocket 전송이 느려지면 Consumer 처리도 같이 느려진다.

그래서 WebSocket 전송을 별도 스레드 풀로 분리했다.

```text
Kafka Consumer 스레드
→ 메시지 수신
→ 위험도 계산
→ DB 저장
→ WebSocket 전송 요청만 맡김
→ 다음 메시지 처리

WebSocket 전용 스레드
→ 실제 WebSocket 전송 처리
```

이렇게 나누면 Consumer는 메시지 처리에 집중하고, WebSocket 전송은 별도 스레드가 처리한다.

---

# 8. AsyncConfig의 역할

`@Async`를 사용하려면 Spring에 비동기 처리를 켜고, 어떤 스레드 풀을 사용할지 알려줘야 한다.

설명용 예시는 다음과 같다.

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Bean(name = "websocketExecutor")
    public Executor websocketExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("ws-async-");

        executor.initialize();
        return executor;
    }

    @Override
    public Executor getAsyncExecutor() {
        return websocketExecutor();
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("비동기 WebSocket 처리 중 예외 발생", ex);
        };
    }
}
```

중요한 설정은 다음과 같다.

```java
@EnableAsync
```

Spring의 비동기 기능을 활성화한다.

```java
executor.setCorePoolSize(5);
```

기본적으로 유지할 스레드 수를 5개로 둔다.

```java
executor.setMaxPoolSize(20);
```

트래픽이 많아졌을 때 최대 20개까지 스레드를 늘릴 수 있게 한다.

```java
executor.setQueueCapacity(100);
```

스레드가 모두 바쁠 때 대기열에 100개까지 작업을 쌓을 수 있게 한다.

```java
executor.setThreadNamePrefix("ws-async-");
```

로그에서 WebSocket 전용 비동기 스레드를 구분하기 쉽게 이름을 붙인다.

```java
AsyncUncaughtExceptionHandler
```

비동기 메서드 안에서 예외가 발생했을 때 로그로 남기기 위한 설정이다.

비동기 메서드는 호출한 쪽에서 예외를 바로 받지 못할 수 있다. 그래서 별도의 예외 처리기가 없으면 문제가 조용히 묻힐 수 있다. Monitory에서는 이 부분까지 고려해서 비동기 예외 로깅을 추가했다.

---

# 9. SensorEventProcessor에서 WebSocket까지의 흐름

센서 데이터가 Kafka로 들어오면 KafkaConsumer가 메시지를 받고, SensorEventProcessor가 실제 처리를 담당한다.

전체 흐름은 다음과 같다.

```text
1. KafkaConsumer가 ENVIRONMENT 토픽 메시지 수신
2. SensorEventProcessor가 센서 값 파싱
3. 위험도 판단
4. 위험 상황이면 AbnormalLog 저장
5. 자동 제어 조건이면 ControlLog 저장
6. AlarmEventService 호출
7. NotificationStrategyFactory가 실행할 알림 전략 선택
8. WebSocketNotificationStrategy 실행
9. WebSocketSender가 /topic/alarm으로 전송
```

중요한 점은 DB 저장과 알림 전송 순서다.

이상 로그와 제어 로그가 저장되기 전에 WebSocket 알림이 먼저 나가면 문제가 생길 수 있다. 프론트는 알림을 받았는데, DB에는 아직 관련 로그가 없을 수 있기 때문이다.

그래서 Monitory에서는 DB 관련 처리를 먼저 끝낸 뒤 알림을 보내도록 순서를 정리했다.

```text
autoControlService.evaluate()
→ DB 관련 처리 완료

alarmEventService.startAlarm()
→ WebSocket 알림 전송
```

이 구조 덕분에 알림은 실제 저장된 이상 로그를 기준으로 전송된다.

---

# 10. NotificationStrategy와 WebSocket

Monitory는 WebSocket만 사용하는 시스템이 아니다.

위험도에 따라 WebSocket, FCM, SMS 같은 여러 알림 채널을 조합할 수 있는 구조를 만들었다.

초기 방식처럼 if문으로 처리하면 이런 코드가 될 수 있다.

```java
if (riskLevel == WARNING) {
    webSocketSender.send(...);
}

if (riskLevel == CRITICAL) {
    webSocketSender.send(...);
    fcmService.send(...);
    smsService.send(...);
}
```

처음에는 단순해 보이지만, 알림 채널이 늘어날수록 조건문이 계속 복잡해진다.

그래서 Monitory에서는 전략 패턴을 사용했다.

```text
NotificationStrategy
├── WebSocketNotificationStrategy
├── AppPushNotificationStrategy
└── SmsNotificationStrategy
```

각 알림 채널은 같은 인터페이스를 구현한다.

```java
public interface NotificationStrategy {
    void send(AlarmEventResponse response);
    RiskLevel getSupportedLevel();
}
```

WebSocket 알림 전략은 다음처럼 볼 수 있다.

```java
@Component
@RequiredArgsConstructor
public class WebSocketNotificationStrategy implements NotificationStrategy {

    private final WebSocketSender webSocketSender;

    @Override
    public void send(AlarmEventResponse response) {
        webSocketSender.sendDangerAlarm(response);
    }

    @Override
    public RiskLevel getSupportedLevel() {
        return RiskLevel.INFO;
    }
}
```

`NotificationStrategyFactory`는 현재 위험도에 맞는 전략 목록을 찾아준다.

```text
WARNING
→ WebSocket

CRITICAL
→ WebSocket + FCM + SMS
```

이 구조의 장점은 새 알림 채널이 추가되어도 기존 서비스 코드를 크게 수정하지 않아도 된다는 점이다.

예를 들어 Slack 알림을 추가하고 싶다면 `SlackNotificationStrategy` 클래스를 하나 추가하면 된다. 기존 `AlarmEventService`는 그대로 둘 수 있다.

---

# 11. 토픽을 나누는 이유

WebSocketSender에는 여러 전송 메서드가 있다.

```text
sendDangerLevel
sendDangerAlarm
sendUnreadCount
sendControlStatus
```

이 메서드들은 서로 다른 토픽으로 메시지를 보낸다.

```text
/topic/zone
/topic/alarm
/topic/unread-count
/topic/control-status
```

이렇게 나누는 이유는 화면마다 필요한 데이터가 다르기 때문이다.

예를 들어 대시보드의 공간 위험도 화면은 `/topic/zone`을 구독하면 되고, 알림 목록은 `/topic/alarm`을 구독하면 된다.

만약 모든 데이터를 하나의 `/topic/all`로 보낸다면 프론트가 메시지를 다시 분류해야 한다. 그러면 프론트 로직이 복잡해지고, 필요 없는 메시지도 계속 받게 된다.

토픽을 나누면 필요한 화면이 필요한 데이터만 받을 수 있다.

---

# 12. WebSocket 구조를 발표에서 설명하는 방법

발표에서는 기술명을 먼저 나열하기보다 문제부터 말하는 것이 좋다.

예시는 다음과 같다.

```text
센서 데이터는 Kafka를 통해 계속 들어오는데,
위험 상황이 발생할 때마다 사용자가 새로고침해서 확인하는 구조는 실시간 모니터링에 맞지 않다고 판단했습니다.

그래서 Kafka Consumer가 이상 상황을 판단하면,
백엔드가 WebSocket을 통해 프론트의 구독 토픽으로 즉시 알림을 전송하도록 구성했습니다.

다만 Consumer 스레드가 WebSocket 전송까지 직접 맡으면,
전송 I/O 때문에 다음 Kafka 메시지 처리가 지연될 수 있습니다.

이를 줄이기 위해 WebSocket 전송 메서드에 @Async를 적용하고,
ThreadPoolTaskExecutor 기반 전용 스레드 풀을 분리했습니다.

결과적으로 Kafka Consumer는 이벤트 처리와 DB 저장에 집중하고,
WebSocket 전송은 별도 스레드에서 처리되는 구조가 되었습니다.
```

더 짧게 말하면 다음과 같다.

```text
Monitory의 WebSocket은 Kafka에서 처리된 위험 이벤트를 화면에 실시간으로 전달하는 역할을 합니다.
저희는 Consumer 스레드가 WebSocket I/O에 묶이지 않도록 전용 비동기 스레드 풀을 분리했고,
위험도별 알림 채널은 전략 패턴으로 분리해 WebSocket, FCM, SMS 등으로 확장 가능한 구조를 만들었습니다.
```

---

# 13. 이 구조에서 중요한 설계 포인트

## 13-1. Consumer 스레드와 전송 스레드를 분리했다

Kafka Consumer는 메시지를 계속 처리해야 한다.
WebSocket 전송은 네트워크 I/O다.

둘을 같은 스레드에서 처리하면 전송 지연이 메시지 처리 지연으로 이어질 수 있다.

그래서 WebSocket 전송을 `@Async`와 전용 스레드 풀로 분리했다.

## 13-2. DB 저장 후 알림을 보냈다

이상 로그가 저장되기 전에 알림을 보내면 프론트에서 상세 조회 시 데이터가 없을 수 있다.

그래서 DB 관련 처리가 끝난 뒤 알림을 보내도록 순서를 정리했다.

## 13-3. 토픽을 목적별로 나눴다

알림, 공간 위험도, 미읽음 카운트, 제어 상태는 서로 다른 데이터다.

각 화면에서 필요한 데이터만 구독할 수 있도록 토픽을 분리했다.

## 13-4. 전략 패턴으로 알림 채널을 분리했다

위험도에 따라 실행할 알림 채널이 달라진다.

조건문이 계속 늘어나는 구조 대신, 각 알림 채널을 전략 클래스로 분리했다.

---

# 14. 헷갈릴 수 있는 개념 정리

## WebSocket

서버와 클라이언트가 연결을 유지하면서 양방향으로 메시지를 주고받는 통신 방식이다.

## STOMP

WebSocket 위에서 사용하는 메시지 규칙이다. `/topic/alarm` 같은 주소를 기준으로 구독과 발행을 할 수 있게 해준다.

## SockJS

WebSocket을 지원하지 않는 환경에서도 비슷하게 동작하도록 도와주는 fallback 라이브러리다.

## SimpMessagingTemplate

Spring에서 STOMP 메시지를 특정 토픽으로 보내는 도구다.

## @Async

메서드를 호출한 스레드에서 바로 실행하지 않고, 별도 스레드에서 실행하게 하는 Spring 기능이다.

## ThreadPoolTaskExecutor

Spring에서 사용하는 스레드 풀 구현체다. 동시에 처리할 스레드 수와 대기열 크기를 설정할 수 있다.

## NotificationStrategy

알림 채널별 실행 로직을 같은 인터페이스로 묶은 구조다. WebSocket, FCM, SMS 같은 채널을 각각 별도 클래스로 분리할 수 있다.

---

# 15. 면접 질문 대비

## Q1. WebSocket을 왜 사용했나요?

공장 모니터링 시스템에서는 위험 상황을 사용자가 새로고침해서 확인하면 늦다고 판단했다. 센서 데이터가 Kafka를 통해 실시간으로 들어오고, 백엔드가 이상 상황을 판단하면 즉시 프론트 화면에 알려야 했다. 그래서 서버가 클라이언트에게 먼저 메시지를 보낼 수 있는 WebSocket을 사용했다.

## Q2. WebSocket과 HTTP Polling의 차이는 무엇인가요?

HTTP Polling은 클라이언트가 일정 주기로 서버에 요청을 보내 최신 데이터를 확인하는 방식이다. 알림이 없어도 계속 요청을 보내야 하고, 요청 주기에 따라 지연이 생길 수 있다. WebSocket은 한 번 연결을 맺고 유지하면서 서버가 필요한 순간 클라이언트에게 바로 메시지를 보낼 수 있다. 실시간 알림에는 WebSocket이 더 적합하다.

## Q3. STOMP는 무엇인가요?

STOMP는 WebSocket 연결 위에서 메시지를 주고받는 규칙이다. WebSocket이 통신 연결이라면, STOMP는 `/topic/alarm` 같은 목적지를 기준으로 메시지를 보내고 구독하는 방식이다. Spring에서는 `SimpMessagingTemplate`을 사용해 특정 토픽으로 메시지를 보낼 수 있다.

## Q4. WebSocket 전송에 @Async를 붙인 이유는 무엇인가요?

Kafka Consumer 스레드가 WebSocket 전송까지 직접 처리하면, 전송 I/O가 지연될 때 다음 Kafka 메시지 처리도 늦어질 수 있다. 그래서 WebSocket 전송을 별도 스레드 풀에서 처리하도록 `@Async`를 적용했다. 이 구조에서는 Consumer는 메시지 처리와 DB 저장에 집중하고, WebSocket 전송은 전용 스레드가 맡는다.

## Q5. 전용 ThreadPoolTaskExecutor를 둔 이유는 무엇인가요?

기본 비동기 실행기를 사용하면 어떤 비동기 작업이 어떤 스레드에서 처리되는지 추적하기 어렵고, 다른 비동기 작업과 자원을 공유하게 된다. WebSocket 전송은 실시간 알림과 관련된 중요한 I/O 작업이므로 별도 스레드 풀을 두어 처리량과 대기열을 관리했다.

## Q6. 비동기 처리에서 예외는 어떻게 처리해야 하나요?

비동기 메서드는 호출한 쪽에서 예외를 바로 받기 어렵다. 그래서 `AsyncUncaughtExceptionHandler`를 등록해 비동기 메서드 내부에서 발생한 예외를 로그로 남기도록 해야 한다. 그렇지 않으면 WebSocket 전송 실패가 조용히 묻힐 수 있다.

## Q7. 알림을 DB 저장 전에 보내면 어떤 문제가 생길 수 있나요?

프론트가 알림을 받고 상세 로그를 조회했는데, 아직 DB에 이상 로그가 저장되지 않았을 수 있다. 그러면 사용자는 알림은 봤지만 상세 정보는 조회하지 못하는 상태가 된다. 그래서 DB 저장과 자동 제어 처리가 끝난 뒤 WebSocket 알림을 보내도록 순서를 정리했다.

## Q8. NotificationStrategy 패턴을 사용한 이유는 무엇인가요?

위험도에 따라 실행해야 하는 알림 채널이 다르기 때문이다. 조건문으로 WebSocket, FCM, SMS를 모두 처리하면 새 채널이 추가될 때 기존 분기 코드를 계속 수정해야 한다. 전략 패턴을 사용하면 새 알림 채널은 새로운 Strategy 클래스로 추가하면 되고, 기존 알림 서비스 구조는 크게 바꾸지 않아도 된다.

## Q9. WebSocket 토픽을 여러 개로 나눈 이유는 무엇인가요?

화면마다 필요한 데이터가 다르기 때문이다. 위험 알림은 `/topic/alarm`, 공간 위험도는 `/topic/zone`, 미읽음 카운트는 `/topic/unread-count`처럼 나누면 프론트가 필요한 토픽만 구독할 수 있다. 이렇게 하면 불필요한 메시지 수신을 줄이고 프론트 로직도 단순해진다.

## Q10. 이 구조의 한계는 무엇인가요?

현재 WebSocket 전송이 서버 메모리와 연결 상태에 의존하기 때문에, 서버가 여러 대로 늘어나면 세션 공유나 메시지 브로커 연동이 필요할 수 있다. 예를 들어 Redis Pub/Sub이나 외부 메시지 브로커를 사용해 여러 서버 인스턴스 간 WebSocket 메시지를 동기화하는 구조를 고려할 수 있다.

---

# 16. 최종 정리

Monitory의 WebSocket 구조는 단순히 화면에 알림을 띄우는 기능이 아니다.

Kafka로 들어온 실시간 센서 이벤트를 백엔드에서 처리하고, 위험 상황을 판단한 뒤, 프론트 화면에 즉시 전달하기 위한 실시간 알림 파이프라인이다.

핵심은 세 가지다.

첫째, Kafka Consumer와 WebSocket 전송을 분리했다.
Consumer 스레드가 WebSocket I/O에 묶이지 않도록 `@Async`와 전용 스레드 풀을 사용했다.

둘째, DB 저장 후 알림을 보냈다.
이상 로그와 제어 로그가 안정적으로 저장된 뒤 WebSocket 알림을 전송하도록 처리 순서를 정리했다.

셋째, 알림 채널을 전략 패턴으로 분리했다.
WebSocket, FCM, SMS 같은 알림 채널을 독립된 Strategy로 나누어 확장 가능한 구조를 만들었다.

따라서 이 구조는 실시간성, 처리 안정성, 확장성을 함께 고려한 WebSocket 알림 설계라고 설명할 수 있다.