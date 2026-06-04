# Chapter 13: Microservices Architecture

---

## 13.1 From Monolith to Distributed Systems

Every application we have built in this course has been a **monolith** — a single Spring Boot application that packages the UI, business logic, and data access into one deployable JAR. The controllers, services, and repositories all share the same process, the same memory, and the same database. This is not a flaw. Monoliths are simple to develop, simple to test, and simple to deploy. For most applications, they are the right choice.

But monoliths have limits. As the codebase grows, a change to the payment module requires redeploying the entire application. Scaling the search functionality means scaling everything else with it. A memory leak in the notification system brings down the order processing. When an application reaches a certain size — in team count, in traffic, in complexity — these constraints start to matter.

**Microservices** are an architectural style that addresses these constraints by decomposing an application into small, independent services. Each service runs in its own process, owns its own data, and communicates with other services over the network. This chapter introduces the concepts, trade-offs, and Spring Cloud tools that make microservices possible. We will build a working multi-service system from scratch to see how the pieces connect.

---

## 13.2 Monolith vs. Microservices

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | Single unit — deploy the whole application. | Each service deployed independently. |
| **Scaling** | Scale the entire application. | Scale individual services. |
| **Technology** | Single tech stack. | Each service can use a different stack. |
| **Data management** | Shared database. | Each service owns its data. |
| **Communication** | In-process method calls. | Network calls (HTTP, messaging). |
| **Failure isolation** | One bug can bring down everything. | Failures are contained per service. |
| **Complexity** | Simple to start, harder as it grows. | Higher operational complexity from day one. |
| **Testing** | Easier end-to-end testing. | Requires integration and contract tests. |
| **Development speed** | Fast initially, slows with growth. | Slower start, scales better long-term. |

### When to Use a Monolith

Start with a monolith when the team is small (1–5 developers), the requirements are evolving, the domain is not well understood yet, or you need to ship quickly. The monolith is not a stepping stone — it is a legitimate, permanent architecture for many applications.

### When to Consider Microservices

Consider microservices when the team is large enough that independent deployment matters, when specific components have very different scaling requirements, when the domain is well understood and can be cleanly decomposed, or when you need to deploy parts of the system independently at different cadences.

The most important advice in this chapter: **do not start with microservices.** Start with a well-structured monolith. Decompose only when the pain of the monolith exceeds the pain of distributed systems. Premature decomposition is one of the most expensive architectural mistakes a team can make.

---

## 13.3 Decomposing a Monolith

If you do decide to decompose, the key challenge is finding the right boundaries. Domain-Driven Design (DDD) provides useful concepts:

**Bounded Context** — each microservice should represent a self-contained area of the business domain with clear boundaries. An order is an order, a product is a product, and they are managed by different services.

**Single Responsibility** — each service does one thing well.

**Data Ownership** — each service owns and manages its own data. No shared databases.

Consider a monolithic e-commerce application decomposed into services:

| Service | Responsibility | Database |
|---------|---------------|----------|
| User Service | Authentication, profiles | Users DB |
| Product Service | Catalog, search, categories | Products DB |
| Order Service | Cart, checkout, order history | Orders DB |
| Payment Service | Payment processing | Payments DB |
| Notification Service | Emails, SMS, push notifications | Notifications DB |

The decomposition follows a gradual strategy known as the **Strangler Fig Pattern** — extract one module at a time, replace internal method calls with REST or messaging, set up a separate database, and keep the system running throughout the transition.

---

## 13.4 Inter-Service Communication

In a monolith, components call each other through method invocations — fast, reliable, and type-safe. In a microservices architecture, components communicate over the network. This introduces latency, failure modes, and serialization overhead. There are two approaches:

### Synchronous Communication (REST)

Services call each other directly using HTTP. The caller waits for the response. This is simple and appropriate when the caller needs the result immediately:

```java
@Service
public class OrderService {

    private final RestClient restClient;

    public OrderService(RestClient.Builder builder) {
        this.restClient = builder
                .baseUrl("http://product-service")
                .build();
    }

    public ProductDTO getProduct(Long productId) {
        return restClient.get()
                .uri("/api/products/{id}", productId)
                .retrieve()
                .body(ProductDTO.class);
    }
}
```

Synchronous communication is easy to understand and debug, but it creates tight coupling — if the product service is down, the order service fails too.

### Asynchronous Communication (Messaging)

Services communicate through a message broker (ActiveMQ, RabbitMQ, Kafka — as covered in Chapter 12). The producer sends a message and does not wait for a response:

```java
@Service
public class OrderService {

    private final JmsTemplate jmsTemplate;

    public OrderService(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }

    public OrderDTO createOrder(OrderRequest request) {
        Order saved = orderRepository.save(new Order(request));
        jmsTemplate.convertAndSend("order-events",
                new OrderCreatedEvent(saved.getId(), saved.getUserId()));
        return new OrderDTO(saved);
    }
}
```

Asynchronous communication provides loose coupling, better resilience (messages queue up if a consumer is down), and better scalability (multiple consumers process in parallel). The trade-off is complexity and eventual consistency.

### When to Use Which

| Use Case | Approach |
|----------|----------|
| Fetching data the caller needs immediately. | Synchronous (REST) |
| Triggering background work (email, logging). | Asynchronous (Messaging) |
| Notifying multiple services about an event. | Asynchronous (Messaging) |
| User-facing operations requiring immediate response. | Synchronous (REST) |

---

## 13.5 The Spring Cloud Ecosystem

Spring Cloud provides tools for the common challenges of distributed systems. It builds on Spring Boot and integrates with the infrastructure components that microservices need.

Spring Cloud **2025.1** (codename "Oakwood") is the release compatible with Spring Boot 4 and Spring Framework 7. All examples in this chapter use this version. Add the BOM to your `pom.xml`:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2025.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

The key components:

| Component | Purpose |
|-----------|---------|
| **Eureka** (Service Discovery) | Services register themselves and discover each other by name. |
| **Spring Cloud Gateway** | Single entry point that routes requests to the right service. |
| **Spring Cloud Config** | Centralized configuration stored in Git, served to all services. |
| **Resilience4j** | Circuit breakers, retries, and rate limiting for fault tolerance. |
| **Spring Cloud LoadBalancer** | Client-side load balancing when multiple instances of a service exist. |

---

## 13.6 A Working Example: The Calculator System

To see how microservices connect in practice, we will build a minimal three-service system. The business logic is deliberately trivial — basic arithmetic — so you can focus on how the services talk to each other rather than what they compute.

The architecture:

```
┌──────────────────┐
│  discovery-server │  (Eureka registry — port 8761)
└──────────────────┘
        ▲       ▲
        │       │
   register  register
        │       │
┌───────┴──┐  ┌─┴────────────┐
│calculator │  │ math-facade   │
│  service  │  │   service     │
│(port 8081)│  │ (port 8082)   │
└───────────┘  └───────────────┘
                    │
              calls via
           service discovery
                    │
              ┌─────┴─────┐
              │ calculator │
              │  service   │
              └────────────┘
```

**discovery-server** — the Eureka registry where services register themselves.

**calculator-service** — a stateless service that performs arithmetic operations.

**math-facade-service** — the user-facing service that accepts requests, calls the calculator service through service discovery, and keeps a history of calculations.

### 13.6.1 The Discovery Server

Create a Spring Boot application with the Eureka Server dependency:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(DiscoveryServerApplication.class, args);
    }
}
```

```properties
server.port=8761
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

The Eureka Server does not register itself (it *is* the registry). Start it first — it provides the dashboard at `http://localhost:8761` where you can see which services have registered.

### 13.6.2 The Calculator Service

Create a second Spring Boot application with the web starter and Eureka client:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```properties
spring.application.name=calculator-service
server.port=8081
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

```java
@RestController
@RequestMapping("/math")
public class CalculatorController {

    private static final Logger log = LoggerFactory.getLogger(CalculatorController.class);

    @GetMapping("/add")
    public int add(@RequestParam int a, @RequestParam int b) {
        log.info("calculator-service handling: {} + {}", a, b);
        return a + b;
    }

    @GetMapping("/subtract")
    public int subtract(@RequestParam int a, @RequestParam int b) {
        log.info("calculator-service handling: {} - {}", a, b);
        return a - b;
    }
}
```

No `@EnableEurekaClient` annotation is needed — Spring Boot auto-configures the Eureka client when the dependency is on the classpath. The `spring.application.name` property is critical: it is the **logical name** that other services use to find this one.

### 13.6.3 The Math Facade Service

Create a third Spring Boot application with the web starter, Eureka client, and the load balancer:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

```properties
spring.application.name=math-facade-service
server.port=8082
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

**The RestClient configuration** — this is the key integration point. To make `RestClient` resolve Eureka service names (like `http://calculator-service`) instead of hardcoded URLs, the `RestClient.Builder` must be `@LoadBalanced`:

```java
@Configuration
public class RestClientConfig {

    @Bean
    @LoadBalanced
    public RestClient.Builder restClientBuilder() {
        return RestClient.builder();
    }

    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder.baseUrl("http://calculator-service").build();
    }
}
```

`@LoadBalanced` is the crucial annotation. Without it, `http://calculator-service` would be treated as a literal hostname and fail with a DNS error. With it, Spring Cloud LoadBalancer intercepts the request, queries Eureka for instances of `calculator-service`, and routes the request to one of them.

**The controller:**

```java
@RestController
@RequestMapping("/api")
public class FacadeController {

    private final RestClient restClient;
    private final List<String> history = new CopyOnWriteArrayList<>();

    public FacadeController(RestClient restClient) {
        this.restClient = restClient;
    }

    @GetMapping("/calculate")
    public String calculate(@RequestParam int a,
                            @RequestParam int b,
                            @RequestParam(defaultValue = "add") String op) {
        Integer result = restClient.get()
                .uri(uriBuilder -> uriBuilder
                        .path("/math/" + op)
                        .queryParam("a", a)
                        .queryParam("b", b)
                        .build())
                .retrieve()
                .body(Integer.class);

        String record = String.format("%d %s %d = %d", a, op, b, result);
        history.add(record);
        return "Result from calculator-service: " + record;
    }

    @GetMapping("/history")
    public List<String> getHistory() {
        return history;
    }
}
```

### 13.6.4 Running the System

Start the three applications in order:

1. **discovery-server** (wait a few seconds for it to initialize).
2. **calculator-service**.
3. **math-facade-service**.

Verify:

1. Open `http://localhost:8761` — the Eureka dashboard shows both `CALCULATOR-SERVICE` and `MATH-FACADE-SERVICE` registered.
2. Call `http://localhost:8082/api/calculate?a=10&b=25` — the facade calls the calculator through service discovery and returns the result.
3. Check the calculator-service console — it logs the request, proving the HTTP call traveled between two separate JVM processes.
4. Call `http://localhost:8082/api/history` — the facade maintains its own state independently.

This is the essence of microservices: independent applications communicating over the network, discovered dynamically through a registry.

---

## 13.7 API Gateway

In the calculator example, the client calls the facade service directly. In a real system with dozens of services, exposing each service's URL to clients is impractical. An **API Gateway** provides a single entry point that routes requests to the correct backend service.

Spring Cloud Gateway handles routing, load balancing, authentication, rate limiting, and request transformation:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
server:
  port: 8080

spring:
  cloud:
    gateway:
      routes:
        - id: calculator-service
          uri: lb://calculator-service
          predicates:
            - Path=/math/**

        - id: math-facade-service
          uri: lb://math-facade-service
          predicates:
            - Path=/api/**
```

The `lb://` prefix tells the gateway to use Eureka for service discovery and load balance across instances. Clients now send all requests to `http://localhost:8080` — the gateway routes `/math/**` requests to the calculator service and `/api/**` requests to the facade service.

---

## 13.8 Centralized Configuration

In a microservices system, managing `application.properties` across many services is error-prone. Spring Cloud Config provides a centralized configuration server that stores configuration in a Git repository and serves it to all services at startup.

### Config Server

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
```

### Config Client

Each service imports its configuration from the config server:

```yaml
spring:
  application:
    name: calculator-service
  config:
    import: configserver:http://localhost:8888
```

The Config Server serves a file named `calculator-service.yml` from the Git repository. When the configuration changes, services can refresh without restarting. This keeps all environment-specific values in one place, version-controlled, and auditable — the same benefits we discussed for property files in Chapter 5, but centralized across an entire system.

---

## 13.9 Resilience with Circuit Breakers

In a distributed system, failures are inevitable. A circuit breaker prevents cascading failures by stopping calls to a failing service and providing a fallback response.

The circuit breaker has three states: **closed** (normal — requests pass through), **open** (failing — requests are immediately rejected with a fallback after a failure threshold is reached), and **half-open** (recovery — a few test requests are allowed through to check if the service has recovered).

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

Apply a circuit breaker to the facade service's call to the calculator:

```java
@Service
public class CalculatorClient {

    private final RestClient restClient;

    public CalculatorClient(RestClient restClient) {
        this.restClient = restClient;
    }

    @CircuitBreaker(name = "calculatorService", fallbackMethod = "addFallback")
    public int add(int a, int b) {
        return restClient.get()
                .uri(uriBuilder -> uriBuilder
                        .path("/math/add")
                        .queryParam("a", a)
                        .queryParam("b", b)
                        .build())
                .retrieve()
                .body(Integer.class);
    }

    public int addFallback(int a, int b, Throwable throwable) {
        // Return a safe default when the calculator service is unreachable
        return 0;
    }
}
```

Configure the circuit breaker behavior in `application.yml`:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      calculatorService:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
```

This configuration opens the circuit after 50% of the last 10 calls fail, waits 10 seconds before allowing test requests, and permits 3 test calls in the half-open state. If the test calls succeed, the circuit closes and normal traffic resumes.

Resilience4j also provides **retry** (automatically retry failed calls), **rate limiter** (limit call frequency), **bulkhead** (limit concurrent calls), and **time limiter** (set timeouts). These patterns are essential in distributed systems where network failures, slow services, and resource exhaustion are routine rather than exceptional.

---

## 13.10 The Complete Architecture

Here is what a complete microservices architecture looks like with all Spring Cloud components:

```
                      ┌──────────────┐
                      │    Clients   │
                      └──────┬───────┘
                             │
                      ┌──────┴───────┐
                      │  API Gateway │
                      │ (Spring Cloud│
                      │   Gateway)   │
                      └──────┬───────┘
                             │
           ┌─────────────────┼──────────────────┐
           │                 │                   │
    ┌──────┴──────┐   ┌──────┴──────┐   ┌───────┴──────┐
    │User Service  │   │Order Service │   │Product       │
    │(Spring Boot) │   │(Spring Boot) │   │Service       │
    └──────┬──────┘   └──────┬──────┘   └───────┬──────┘
           │                 │                   │
           │          ┌──────┴───────┐           │
           │          │Message Broker│           │
           │          │ (ActiveMQ)   │           │
           │          └──────────────┘           │
           │                                     │
    ┌──────┴──────┐                      ┌───────┴──────┐
    │Eureka Server│                      │Config Server │
    │ (Registry)  │                      │  (Git-based) │
    └─────────────┘                      └──────────────┘
```

The request flow: a client sends a request to the **API Gateway**, which consults **Eureka** to discover the target service and routes the request accordingly. Services call each other using **RestClient** (synchronous) or a **message broker** (asynchronous). **Resilience4j** circuit breakers protect against cascading failures. All services pull their configuration from the **Config Server** at startup.

---

## Summary

This chapter introduced microservices architecture and the Spring Cloud ecosystem.

**Monoliths vs. microservices** is not a good-vs-bad choice. Monoliths are simpler, faster to develop, and easier to test. Microservices provide independent deployment, independent scaling, and failure isolation — at the cost of distributed system complexity. Start with a monolith; decompose when the need is clear.

**Service boundaries** follow Domain-Driven Design principles: bounded contexts, single responsibility, and data ownership. The Strangler Fig Pattern enables gradual decomposition.

**Inter-service communication** is either synchronous (REST with `RestClient`) or asynchronous (messaging with JMS, RabbitMQ, or Kafka). Synchronous is simpler; asynchronous is more resilient. Most systems use both.

**Service Discovery** with Eureka lets services register themselves and find each other by logical name rather than hardcoded URLs. `@LoadBalanced` on `RestClient.Builder` enables service-name resolution.

**API Gateway** with Spring Cloud Gateway provides a single entry point for clients, handling routing, load balancing, and cross-cutting concerns.

**Centralized Configuration** with Spring Cloud Config stores configuration for all services in a Git repository, enabling consistent, version-controlled management.

**Circuit Breakers** with Resilience4j prevent cascading failures by stopping calls to failing services and providing fallback responses.

**Spring Cloud 2025.1** is the release compatible with Spring Boot 4 and Spring Framework 7.

---

## Resources

- [Spring Cloud Official Documentation](https://spring.io/projects/spring-cloud)
- [Spring Cloud Netflix Eureka](https://spring.io/projects/spring-cloud-netflix)
- [Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway)
- [Spring Cloud Config](https://spring.io/projects/spring-cloud-config)
- [Resilience4j Documentation](https://resilience4j.readme.io/)
- [Microservices Patterns — Chris Richardson](https://microservices.io/patterns/)
- [Martin Fowler: Microservices](https://martinfowler.com/articles/microservices.html)
- [Baeldung: Spring Cloud Series](https://www.baeldung.com/spring-cloud-series)

---

## Lab Assignment: Design and Build a Minimal Microservices System

This week focuses on concepts and a working demo rather than a full production system. The lab has two parts.

### Part 1: Architecture Design

Choose one of the following application scenarios: an online bookstore (users, catalog, orders, reviews, notifications), a food delivery platform (users, restaurants, menus, orders, delivery tracking), or a university management system (students, courses, enrollment, grades, notifications).

For your chosen scenario:

1. **Identify 4–5 microservices** and describe the responsibility of each.
2. **Draw an architecture diagram** showing each service and its database, the API Gateway, the service registry (Eureka), and communication paths — mark which are synchronous REST and which are asynchronous messaging.
3. **Define 3–4 REST API endpoints** for each service (method, path, request/response).
4. **Identify at least one scenario** where asynchronous messaging is more appropriate than synchronous REST, and explain why.

Submit the diagram and descriptions as a document.

### Part 2: Minimal Implementation

Build two minimal Spring Boot services that communicate via REST:

1. **Service A** (e.g., Order Service) — exposes a `POST /api/orders` endpoint.
2. **Service B** (e.g., Product Service) — exposes a `GET /api/products/{id}` endpoint.

When Service A receives an order request, it calls Service B using `RestClient` to fetch product details, then returns a combined response.

**Requirements:**
- Two separate Spring Boot projects, each with its own `pom.xml` and `application.properties`.
- Each service runs on a different port.
- Service A calls Service B using `RestClient`.
- Proper error handling if Service B is unavailable.

**Bonus:**
- Add a Eureka Server and register both services with it. Use logical service names instead of hardcoded URLs.
- Add a circuit breaker with Resilience4j on Service A's call to Service B, with a fallback method.
