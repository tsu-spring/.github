# Chapter 12: Messaging and Asynchronous Processing

---

## 12.1 Beyond Request-Response

Every interaction in the previous chapters followed the same pattern: a client sends a request, the server processes it, and the server sends a response. The client waits for the entire operation to complete. This is **synchronous** processing — straightforward, easy to reason about, and perfectly adequate for most web operations.

But some tasks do not fit this model. Sending a confirmation email after an order is placed should not make the user wait 3 seconds for the SMTP server to respond. Processing a large file upload should not block the HTTP thread for 30 seconds. Notifying a dozen downstream services about a price change should not happen inline with the API call that triggered it.

This chapter introduces **asynchronous processing** — running tasks in the background so the caller does not wait — and **messaging** — communicating between components or systems through a message broker rather than direct method calls. We will learn how to run background tasks with `@Async`, publish and listen to in-application events, send and receive messages through JMS with ActiveMQ, schedule recurring tasks with `@Scheduled`, send emails with `JavaMailSender`, and implement real-time communication with WebSockets.

---

## 12.2 Synchronous vs. Asynchronous Processing

| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| Execution | Blocking — the caller waits for the operation to complete. | Non-blocking — the caller continues immediately. |
| Response | The result is available when the call returns. | The result arrives later (or there is no result). |
| Thread usage | One thread is occupied for the duration of the call. | The work happens on a separate thread. |
| Example | A REST API that queries a database and returns the result. | Sending an email after a user registers. |

Asynchronous processing is appropriate when a task takes a long time to complete (sending emails, generating reports, calling slow external services), when the caller does not need the result immediately, or when you want to improve throughput by freeing up request-handling threads.

---

## 12.3 Messaging Concepts

Messaging is a communication pattern where components exchange data through a central intermediary — a **message broker** — rather than calling each other directly.

The pattern has three participants: a **producer** creates a message and sends it to the broker, the **broker** stores the message and routes it to the right destination, and a **consumer** retrieves the message and processes it. The producer and consumer do not need to be running at the same time, do not need to know about each other, and do not need to be written in the same language.

Messaging systems use two delivery models:

| Model | Behavior | Analogy |
|-------|----------|---------|
| **Queue** | One message is delivered to exactly one consumer. If multiple consumers listen on the same queue, messages are distributed in round-robin. | A letter — one recipient reads it. |
| **Topic** | One message is delivered to all subscribers. Every consumer listening on the topic receives a copy. | A broadcast — everyone tuned in hears it. |

### Messaging Options in Spring Boot

| Tool | Type | Best For |
|------|------|----------|
| `@Async` | In-process background threads | Simple async logic within one application. |
| Spring Events | In-process event publishing | Decoupling components within one application. |
| JMS (ActiveMQ) | External message broker | Enterprise systems, reliable queuing, Java-to-Java. |
| RabbitMQ | External message broker (AMQP) | Reliable queues, microservices. |
| Kafka | Distributed streaming platform | High-throughput event streaming, log aggregation. |

This chapter covers `@Async`, Spring Events, and JMS with ActiveMQ in depth. RabbitMQ and Kafka follow similar integration patterns — Spring Boot provides starters for both — but their setup and operational characteristics differ and are beyond this chapter's scope.

---

## 12.4 Background Tasks with @Async

The `@Async` annotation runs a method on a separate thread. The caller returns immediately without waiting for the method to complete.

### Enabling Async Support

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}
```

`@EnableAsync` activates Spring's asynchronous method execution support. Without it, `@Async` annotations on methods are silently ignored.

### Fire-and-Forget Methods

The simplest async pattern is a void method — fire the task and forget about it:

```java
@Service
public class NotificationService {

    private static final Logger log = LoggerFactory.getLogger(NotificationService.class);

    @Async
    public void sendNotification(String userId, String message) {
        log.info("Sending notification on thread: {}", Thread.currentThread().getName());
        // Simulate slow operation
        try { Thread.sleep(3000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        log.info("Notification sent to user: {}", userId);
    }
}
```

When a controller calls `notificationService.sendNotification(...)`, the method returns immediately and the actual work happens on a background thread. The user's HTTP response is not delayed.

### Methods That Return a Result

When you need the result of an async operation, return a `CompletableFuture`:

```java
@Async
public CompletableFuture<String> fetchDataFromExternalService() {
    log.info("Fetching data on thread: {}", Thread.currentThread().getName());
    String result = restClient.get().uri("/data").retrieve().body(String.class);
    return CompletableFuture.completedFuture(result);
}
```

`CompletableFuture.completedFuture(result)` wraps the result in a `CompletableFuture` that is already resolved. The caller can check if the future is done, wait for the result, or compose it with other futures.

Note that `AsyncResult` (which older tutorials and documentation may reference) has been deprecated since Spring Framework 6.0. Always use `CompletableFuture` instead.

### Combining Multiple Async Results

`CompletableFuture` supports combining results from multiple async calls:

```java
@Service
public class AggregationService {

    private final UserService userService;
    private final OrderService orderService;

    public AggregationService(UserService userService, OrderService orderService) {
        this.userService = userService;
        this.orderService = orderService;
    }

    public DashboardData getDashboardData(Long userId) throws Exception {
        CompletableFuture<UserProfile> profileFuture = userService.getProfileAsync(userId);
        CompletableFuture<List<Order>> ordersFuture = orderService.getOrdersAsync(userId);

        // Wait for both to complete
        CompletableFuture.allOf(profileFuture, ordersFuture).join();

        return new DashboardData(profileFuture.get(), ordersFuture.get());
    }
}
```

Both async calls execute concurrently. `CompletableFuture.allOf(...).join()` blocks until both are complete. If the profile fetch takes 2 seconds and the orders fetch takes 3 seconds, the total wait is 3 seconds — not 5.

### Important Limitations

**`@Async` only works on public methods.** Spring implements `@Async` through proxying. A proxy can only intercept calls that come through the proxy — which means calls from outside the class. Private methods and self-invocations (calling an `@Async` method from within the same class) bypass the proxy and execute synchronously.

**Do not call an `@Async` method from the same class.** This is the most common mistake. If `ServiceA.methodA()` calls `ServiceA.asyncMethodB()`, the async annotation is ignored because the call does not go through the proxy. Move the async method to a separate service.

### Customizing the Thread Pool

By default, Spring uses `SimpleAsyncTaskExecutor`, which creates a new thread for every task. For production, configure a proper thread pool:

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

Reference the executor by name if you have multiple:

```java
@Async("taskExecutor")
public void processInBackground() {
    // Runs on the "taskExecutor" thread pool
}
```

### Exception Handling for Void Async Methods

When a `CompletableFuture`-returning async method throws an exception, the caller sees it when calling `.get()`. But for void methods, the exception has nowhere to go. Implement `AsyncUncaughtExceptionHandler` to catch them:

```java
public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(CustomAsyncExceptionHandler.class);

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        log.error("Async exception in method {}: {}", method.getName(), ex.getMessage(), ex);
    }
}
```

Register it by implementing `AsyncConfigurer`:

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }
}
```

---

## 12.5 Spring Application Events

Spring Events provide a way for components within the same application to communicate without depending on each other directly. One component publishes an event, and any number of listeners react to it.

### Defining an Event

```java
public class UserRegisteredEvent extends ApplicationEvent {

    private final String email;

    public UserRegisteredEvent(Object source, String email) {
        super(source);
        this.email = email;
    }

    public String getEmail() { return email; }
}
```

### Publishing an Event

Inject `ApplicationEventPublisher` and call `publishEvent`:

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;

    public UserService(UserRepository userRepository,
                       ApplicationEventPublisher eventPublisher) {
        this.userRepository = userRepository;
        this.eventPublisher = eventPublisher;
    }

    public void registerUser(User user) {
        userRepository.save(user);
        eventPublisher.publishEvent(new UserRegisteredEvent(this, user.getEmail()));
    }
}
```

### Listening for Events

```java
@Component
public class WelcomeEmailListener {

    private final EmailService emailService;

    public WelcomeEmailListener(EmailService emailService) {
        this.emailService = emailService;
    }

    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        emailService.sendWelcomeEmail(event.getEmail());
    }
}
```

The `UserService` does not know about the `WelcomeEmailListener`. It simply publishes the event. If you later add a second listener that creates a default profile, or a third that sends an analytics event, the `UserService` does not change. This is the Observer pattern — events decouple the publisher from the subscribers.

By default, `@EventListener` methods run synchronously on the same thread. To process events asynchronously, add `@Async`:

```java
@Async
@EventListener
public void handleUserRegistered(UserRegisteredEvent event) {
    emailService.sendWelcomeEmail(event.getEmail());
}
```

---

## 12.6 Message Brokers and JMS

When you need messaging between separate applications — or when messages must survive application restarts — you need an external message broker.

**JMS (Java Message Service)** is a standard API for sending, receiving, and processing messages in Java. **Spring JMS** simplifies working with JMS through `JmsTemplate` (for sending) and `@JmsListener` (for receiving).

**Apache ActiveMQ** is a popular JMS-compatible broker. Spring Boot provides starter dependencies for both ActiveMQ Classic (5.x) and ActiveMQ Artemis.

### Setup

Add the ActiveMQ starter and broker dependencies:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>

<!-- Embedded broker for development -->
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-broker</artifactId>
    <scope>runtime</scope>
</dependency>
```

For development, the embedded broker runs inside your application — no external installation required. Configure it in `application.properties`:

```properties
spring.activemq.broker-url=vm://localhost?broker.persistent=false
spring.activemq.packages.trust-all=true
```

### Sending Messages with JmsTemplate

```java
@Service
public class OrderProducer {

    private final JmsTemplate jmsTemplate;

    public OrderProducer(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }

    public void sendOrder(Order order) {
        jmsTemplate.convertAndSend("order-queue", order);
        log.info("Sent order to queue: {}", order.getId());
    }
}
```

`convertAndSend` serializes the object and sends it to the named destination. By default, Java serialization is used, which requires the object to implement `Serializable`.

### Receiving Messages with @JmsListener

```java
@Component
public class OrderConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderConsumer.class);

    @JmsListener(destination = "order-queue")
    public void processOrder(Order order) {
        log.info("Received order: {}", order.getId());
        // Process the order
    }
}
```

`@JmsListener` automatically listens on the specified queue. When a message arrives, Spring deserializes it and calls the method. If multiple consumers listen on the same queue, messages are distributed in round-robin.

### JSON Serialization

Java serialization is fragile — class changes can break deserialization, and it only works between Java applications. JSON is more robust and interoperable. Configure a JSON message converter:

```java
@Configuration
public class JmsConfig {

    @Bean
    public MappingJackson2MessageConverter jacksonJmsMessageConverter() {
        MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
        converter.setTargetType(MessageType.TEXT);
        converter.setTypeIdPropertyName("_type");
        return converter;
    }
}
```

With this configuration, messages are sent as JSON text. The `_type` property header tells the receiver which Java class to deserialize into.

### Topics (Publish-Subscribe)

To use the topic model instead of queues, configure the listener factory and template with `setPubSubDomain(true)`:

```java
@Configuration
@EnableJms
public class JmsTopicConfig {

    @Bean
    public DefaultJmsListenerContainerFactory topicListenerFactory(
            ConnectionFactory connectionFactory,
            MappingJackson2MessageConverter messageConverter) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setPubSubDomain(true);
        factory.setMessageConverter(messageConverter);
        return factory;
    }

    @Bean("topicJmsTemplate")
    public JmsTemplate topicJmsTemplate(ConnectionFactory connectionFactory,
                                         MappingJackson2MessageConverter messageConverter) {
        JmsTemplate template = new JmsTemplate(connectionFactory);
        template.setPubSubDomain(true);
        template.setMessageConverter(messageConverter);
        return template;
    }
}
```

On the listener side, reference the topic factory:

```java
@JmsListener(destination = "price-updates", containerFactory = "topicListenerFactory")
public void handlePriceUpdate(PriceUpdate update) {
    log.info("Received price update: {}", update);
}
```

Every consumer subscribed to the `price-updates` topic receives every message — unlike a queue, where each message goes to only one consumer.

---

## 12.7 Scheduling Tasks with @Scheduled

Some tasks need to run on a recurring schedule — cleaning up expired sessions, sending daily reports, refreshing cached data. Spring provides `@Scheduled` for this.

### Enabling Scheduling

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {
}
```

### Scheduling Patterns

**Fixed rate** — runs every N milliseconds, regardless of when the previous execution finished:

```java
@Component
public class ScheduledTasks {

    private static final Logger log = LoggerFactory.getLogger(ScheduledTasks.class);

    @Scheduled(fixedRate = 60000)
    public void cleanupExpiredSessions() {
        log.info("Running session cleanup at {}", LocalDateTime.now());
    }
}
```

**Fixed delay** — waits N milliseconds after the previous execution completes before running again:

```java
@Scheduled(fixedDelay = 30000)
public void pollExternalService() {
    // Runs 30 seconds after the previous execution finishes
}
```

**Initial delay** — adds a delay before the first execution:

```java
@Scheduled(fixedRate = 60000, initialDelay = 10000)
public void warmUpCache() {
    // First run after 10 seconds, then every 60 seconds
}
```

**Cron expressions** — provide fine-grained control using cron syntax:

```java
@Scheduled(cron = "0 0 9 * * MON-FRI")
public void sendDailyReport() {
    // Runs at 9:00 AM every weekday
}
```

Common cron patterns: `"0 0 * * * *"` (every hour), `"0 0 9 * * *"` (daily at 9 AM), `"0 0/30 * * * *"` (every 30 minutes), `"0 0 9 * * MON-FRI"` (weekdays at 9 AM).

The difference between `fixedRate` and `fixedDelay` matters when the task takes longer than the interval. With `fixedRate`, the next execution starts N milliseconds after the previous one *started* — executions can overlap if the task is slow. With `fixedDelay`, the next execution starts N milliseconds after the previous one *finished* — no overlap, but the interval between starts varies.

---

## 12.8 Sending Emails with JavaMailSender

Spring Boot provides email support through `spring-boot-starter-mail`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

### SMTP Configuration

```properties
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=${MAIL_USERNAME}
spring.mail.password=${MAIL_PASSWORD}
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

Credentials come from environment variables (Chapter 5) — never hardcode them in property files.

### Sending a Simple Email

```java
@Service
public class EmailService {

    private final JavaMailSender mailSender;

    public EmailService(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }

    public void sendSimpleEmail(String to, String subject, String body) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setTo(to);
        message.setSubject(subject);
        message.setText(body);
        message.setFrom("noreply@example.com");
        mailSender.send(message);
    }
}
```

### Sending HTML Emails with FreeMarker Templates

For rich HTML emails, use FreeMarker to render the email body from a template. Create a template at `src/main/resources/templates/email/welcome.ftlh`:

```html
<html>
<body>
    <h1>Welcome, ${userName}!</h1>
    <p>Thank you for registering on our platform.</p>
    <p>Your account has been created successfully.</p>
    <a href="${activationLink}">Click here to activate your account</a>
</body>
</html>
```

Process the template and send the email:

```java
@Service
public class EmailService {

    private final JavaMailSender mailSender;
    private final FreeMarkerConfigurer freeMarkerConfigurer;

    public EmailService(JavaMailSender mailSender,
                        FreeMarkerConfigurer freeMarkerConfigurer) {
        this.mailSender = mailSender;
        this.freeMarkerConfigurer = freeMarkerConfigurer;
    }

    public void sendWelcomeEmail(String to, String userName, String activationLink)
            throws Exception {
        Template template = freeMarkerConfigurer.getConfiguration()
                .getTemplate("email/welcome.ftlh");

        Map<String, Object> model = Map.of(
                "userName", userName,
                "activationLink", activationLink);

        String htmlContent = FreeMarkerTemplateUtils.processTemplateIntoString(template, model);

        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
        helper.setTo(to);
        helper.setSubject("Welcome to Our Platform!");
        helper.setText(htmlContent, true);
        helper.setFrom("noreply@example.com");
        mailSender.send(message);
    }
}
```

### Sending Emails Asynchronously

Email sending is a classic candidate for `@Async` — the user should not wait for the SMTP transaction:

```java
@Async
public void sendWelcomeEmailAsync(String to, String userName, String activationLink) {
    try {
        sendWelcomeEmail(to, userName, activationLink);
    } catch (Exception e) {
        log.error("Failed to send welcome email to {}: {}", to, e.getMessage());
    }
}
```

---

## 12.9 Real-Time Communication with WebSockets

All communication patterns covered so far are initiated by the client — the client sends a request, the server responds. **WebSockets** provide full-duplex communication over a single TCP connection — both the client and the server can send messages at any time. This enables real-time features like chat applications, live notifications, collaborative editing, and live dashboards.

### Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

### Configuration with STOMP and SockJS

**STOMP** (Simple Text Oriented Messaging Protocol) provides publish-subscribe semantics on top of WebSockets. **SockJS** provides a fallback for browsers that do not support WebSockets.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS();
    }
}
```

`/topic` is the prefix for messages broadcast from the server to subscribed clients. `/app` is the prefix for messages sent from the client to the server (handled by `@MessageMapping`). `/ws` is the WebSocket connection endpoint.

### Message Controller

```java
@Controller
public class ChatController {

    @MessageMapping("/chat.send")
    @SendTo("/topic/messages")
    public ChatMessage sendMessage(@Payload ChatMessage message) {
        return message;
    }
}
```

When a client sends a message to `/app/chat.send`, the `sendMessage` method processes it and the `@SendTo` annotation broadcasts the result to all clients subscribed to `/topic/messages`.

### JavaScript Client

```html
<script src="https://cdn.jsdelivr.net/npm/sockjs-client/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs/lib/stomp.min.js"></script>

<script>
    var socket = new SockJS('/ws');
    var stompClient = Stomp.over(socket);

    stompClient.connect({}, function(frame) {
        console.log('Connected: ' + frame);

        stompClient.subscribe('/topic/messages', function(message) {
            var msg = JSON.parse(message.body);
            console.log('Received: ' + msg.content);
        });
    });

    function sendMessage() {
        stompClient.send("/app/chat.send", {}, JSON.stringify({
            sender: "User1",
            content: "Hello, everyone!",
            type: "CHAT"
        }));
    }
</script>
```

The client connects to the `/ws` endpoint, subscribes to `/topic/messages` to receive broadcasts, and sends messages to `/app/chat.send`. The server-side controller receives the message and broadcasts it back to all subscribers.

---

## Summary

This chapter covered asynchronous processing and messaging patterns in Spring Boot.

**Synchronous vs. asynchronous processing:** synchronous processing blocks the caller until the operation completes; asynchronous processing returns immediately and completes the work in the background.

**`@Async`** runs methods on background threads. It requires `@EnableAsync`, works only on public methods called from outside the class, and should use `CompletableFuture` for return values. Custom thread pools should be configured for production.

**Spring Events** decouple components within an application through the Observer pattern. A publisher sends an event via `ApplicationEventPublisher`, and any `@EventListener` method that matches the event type reacts to it.

**JMS with ActiveMQ** provides reliable messaging between applications through a broker. `JmsTemplate` sends messages, `@JmsListener` receives them. Queues deliver each message to one consumer; topics deliver to all subscribers. JSON serialization via `MappingJackson2MessageConverter` is preferred over Java serialization.

**`@Scheduled`** runs methods on a recurring schedule. `fixedRate` runs at fixed intervals, `fixedDelay` waits after each completion, and cron expressions provide fine-grained control.

**JavaMailSender** sends emails, from simple plain-text messages to rich HTML rendered from FreeMarker templates. Combining email sending with `@Async` prevents blocking the user's request.

**WebSockets** with STOMP and SockJS enable real-time, bidirectional communication between the server and clients — suited for chat, notifications, and live dashboards.

---

## Resources

- [How To Do @Async in Spring — Baeldung](https://www.baeldung.com/spring-async)
- [Spring Events — Baeldung](https://www.baeldung.com/spring-events)
- [Getting Started: Messaging with JMS — Spring Guides](https://spring.io/guides/gs/messaging-jms/)
- [Apache ActiveMQ](https://activemq.apache.org/)
- [Spring Boot Starter Mail — Baeldung](https://www.baeldung.com/spring-email)
- [WebSocket with Spring Boot — Baeldung](https://www.baeldung.com/spring-websockets-sendtouser)
- [Scheduling Tasks — Spring Boot Reference](https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html)

---

## Lab Assignment: Implement Asynchronous Messaging

Add asynchronous processing and messaging to your application.

**Requirements:**

1. **Implement at least one `@Async` method** — for example, sending an email or processing a file upload in the background. Configure `@EnableAsync` and verify that the method runs on a separate thread by logging the thread name.

2. **Set up a message broker** using one of the following: ActiveMQ Classic (embedded or standalone), ActiveMQ Artemis, RabbitMQ, or Kafka. Implement at least one producer that sends messages and one consumer that receives and processes them.

3. **Use JSON serialization** for messages rather than Java serialization. Configure `MappingJackson2MessageConverter` (for JMS) or the equivalent for your chosen broker.

4. **Include logging** to demonstrate that messages are sent and received asynchronously — timestamps and thread names should show that the producer does not wait for the consumer.

**Bonus challenges:**

5. Add a **scheduled task** with `@Scheduled` that periodically sends messages to the queue (e.g., a heartbeat or a periodic data sync).

6. Send an **email notification** when a message is received, using `JavaMailSender` with a FreeMarker HTML template.

7. Implement a **WebSocket endpoint** that broadcasts received messages to connected browser clients in real time.
