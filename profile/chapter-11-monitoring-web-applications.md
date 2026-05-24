# Chapter 11: Monitoring Web Applications

---

## 11.1 From Development to Production

Throughout this course, we have verified our application by running it locally, clicking through pages, and reading log output in the console. This is development. Production is different. In production, you cannot attach a debugger, you cannot watch the console, and you are not the one using the application — thousands of users are, at any hour, and you need to know when something goes wrong before they tell you.

This chapter introduces observability — the tools and practices that let you understand what your application is doing in production. We will learn the three pillars of observability, how to add and configure Spring Boot Actuator, how to work with built-in health checks and metrics endpoints, how to create custom health indicators and business metrics with Micrometer, how to secure Actuator endpoints, how to implement structured logging for production, and how to integrate with Prometheus and Grafana for monitoring dashboards.

---

## 11.2 The Three Pillars of Observability

Observability is the ability to understand the internal state of a system by examining its external outputs. In production, those outputs come in three forms:

| Pillar | What It Tells You | Examples |
|--------|-------------------|----------|
| **Logs** | What happened — textual records of events. | "Order 1234 created by user alice at 10:32:15." |
| **Metrics** | How much, how fast — numeric measurements over time. | Request rate, error count, memory usage, response latency. |
| **Traces** | The path a request took through the system. | A single user request flowing through the API gateway, the order service, the payment service, and the database. |

**Logs** are the most familiar — we have used `log.info()` and `log.error()` throughout this course. They tell you what happened, but they are hard to aggregate and analyze at scale without structure.

**Metrics** are numbers. They answer questions like "how many requests per second is the application handling?" and "what is the 95th percentile response time?" Metrics are cheap to collect, easy to alert on, and the foundation of dashboards.

**Traces** connect the dots across distributed systems. When a single request touches multiple services, a trace ties them together so you can see where time was spent and where failures occurred. Traces are most valuable in microservice architectures.

This chapter focuses primarily on **metrics** and **health checks** through Spring Boot Actuator, and on **structured logging** for production readiness. Distributed tracing is an advanced topic beyond the scope of this course, but Spring Boot 4 provides excellent support for it through OpenTelemetry integration.

---

## 11.3 Adding Spring Boot Actuator

Spring Boot Actuator is a module that adds production-ready monitoring and management features to your application. Add the dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Once added, Actuator exposes endpoints under `/actuator/`. By default, only the `/actuator/health` endpoint is exposed over HTTP. To make other endpoints accessible, you must explicitly include them in your configuration.

### Basic Configuration

```properties
# Expose specific endpoints
management.endpoints.web.exposure.include=health,info,metrics,loggers

# Or expose all endpoints (development only — never in production)
management.endpoints.web.exposure.include=*

# Change the base path (default is /actuator)
management.endpoints.web.base-path=/manage

# Run Actuator on a separate port (useful for separating management traffic)
management.server.port=9090
```

The `management.endpoints.web.exposure.include` property controls which endpoints are accessible over HTTP. In production, expose only what you need — every exposed endpoint is a potential information leak if not properly secured.

---

## 11.4 Built-In Actuator Endpoints

Actuator provides a rich set of endpoints out of the box:

| Endpoint | Description |
|----------|-------------|
| `/actuator/health` | Application health status (UP, DOWN). |
| `/actuator/info` | Arbitrary application metadata. |
| `/actuator/metrics` | Application metrics (memory, CPU, HTTP requests, etc.). |
| `/actuator/env` | Environment properties and configuration values. |
| `/actuator/beans` | All Spring beans in the application context. |
| `/actuator/mappings` | All `@RequestMapping` paths. |
| `/actuator/loggers` | View and change log levels at runtime. |
| `/actuator/threaddump` | Thread dump of the JVM. |
| `/actuator/heapdump` | Heap dump (downloads a binary file). |
| `/actuator/scheduledtasks` | Scheduled tasks information. |
| `/actuator/caches` | Application caches. |

### The /health Endpoint

The health endpoint reports whether the application is running correctly. Spring Boot automatically includes health checks for components it detects — the database, the disk, Redis, mail servers, and more:

```json
{
    "status": "UP",
    "components": {
        "db": {
            "status": "UP",
            "details": {
                "database": "PostgreSQL",
                "validationQuery": "isValid()"
            }
        },
        "diskSpace": {
            "status": "UP",
            "details": {
                "total": 499963174912,
                "free": 250000000000
            }
        }
    }
}
```

By default, the health endpoint shows only the top-level status (`UP` or `DOWN`) without component details. To see the full breakdown:

```properties
management.endpoint.health.show-details=always
# Options: never (default), when-authorized, always
```

In production, use `when-authorized` — only authenticated users with the right role should see internal health details.

The health endpoint is the foundation of **liveness and readiness probes** in container orchestration systems like Kubernetes. Kubernetes periodically calls `/actuator/health` to determine whether to route traffic to an instance or restart it.

### The /info Endpoint

The info endpoint displays application metadata that you define. Populate it through properties:

```properties
management.info.env.enabled=true
info.app.name=My Spring Boot App
info.app.description=A sample application for the course
info.app.version=1.0.0
```

Or include build information automatically from Maven by adding a goal to the `spring-boot-maven-plugin`:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

This generates a `build-info.properties` file at build time that Actuator reads automatically, exposing the artifact name, version, build time, and other metadata.

### The /metrics Endpoint

The metrics endpoint lists all available metric names. To view a specific metric, append its name:

```
GET /actuator/metrics/jvm.memory.used
GET /actuator/metrics/http.server.requests
GET /actuator/metrics/system.cpu.usage
```

Example response:

```json
{
    "name": "jvm.memory.used",
    "description": "The amount of used memory",
    "baseUnit": "bytes",
    "measurements": [
        {
            "statistic": "VALUE",
            "value": 1.5E8
        }
    ]
}
```

Spring Boot automatically collects dozens of metrics — JVM memory and garbage collection, CPU usage, HTTP request counts and latencies, database connection pool stats, and cache hit/miss ratios. These are available immediately with no additional code.

### The /loggers Endpoint

The loggers endpoint lets you view and **change log levels at runtime** without restarting the application:

```bash
# View the current log level for a package
curl http://localhost:8080/actuator/loggers/com.example.myapp

# Change it to DEBUG
curl -X POST http://localhost:8080/actuator/loggers/com.example.myapp \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

This is extremely useful for debugging production issues. You can temporarily enable DEBUG logging for a specific package, observe the detailed output, and then switch it back to INFO — all without a deployment.

---

## 11.5 Custom Health Indicators

The built-in health checks cover Spring Boot's own infrastructure — the database, the disk, the mail server. But your application likely depends on external services that Spring Boot does not know about — a third-party API, a message queue, a file storage service. You can create custom health indicators for these dependencies:

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final RestClient restClient;

    public ExternalApiHealthIndicator(RestClient restClient) {
        this.restClient = restClient;
    }

    @Override
    public Health health() {
        try {
            restClient.get()
                    .uri("https://api.example.com/health")
                    .retrieve()
                    .body(String.class);

            return Health.up()
                    .withDetail("service", "External API")
                    .withDetail("status", "reachable")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("service", "External API")
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

Implement the `HealthIndicator` interface, return `Health.up()` when the dependency is healthy, and `Health.down()` with diagnostic details when it is not. Spring Boot automatically discovers beans that implement `HealthIndicator` and includes them in the `/actuator/health` response:

```json
{
    "status": "UP",
    "components": {
        "externalApi": {
            "status": "UP",
            "details": {
                "service": "External API",
                "status": "reachable"
            }
        }
    }
}
```

The component name (`externalApi`) is derived from the class name by removing the `HealthIndicator` suffix and lowercasing the first letter. If any health indicator reports DOWN, the overall application status becomes DOWN.

---

## 11.6 Custom Metrics with Micrometer

**Micrometer** is the metrics facade used by Spring Boot — similar to how SLF4J is a facade for logging. You write metrics code against Micrometer's API, and Micrometer forwards the data to whichever monitoring system you choose (Prometheus, Datadog, New Relic, InfluxDB, and others).

### Metric Types

| Type | Description | Example |
|------|-------------|---------|
| **Counter** | A value that only increases. | Total number of orders placed. |
| **Gauge** | A value that can go up and down. | Current number of active sessions. |
| **Timer** | Measures both duration and count. | Time spent processing each order. |
| **Distribution Summary** | Like Timer, but for non-time values. | Size of uploaded files. |

### Creating Custom Metrics

Inject the `MeterRegistry` and create metrics in the constructor:

```java
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer orderTimer;
    private final OrderRepository orderRepository;

    public OrderService(MeterRegistry meterRegistry, OrderRepository orderRepository) {
        this.orderRepository = orderRepository;

        this.orderCounter = Counter.builder("orders.created.total")
                .description("Total number of orders created")
                .tag("type", "online")
                .register(meterRegistry);

        this.orderTimer = Timer.builder("orders.processing.time")
                .description("Time to process an order")
                .register(meterRegistry);
    }

    public Order createOrder(Order order) {
        return orderTimer.record(() -> {
            Order saved = orderRepository.save(order);
            orderCounter.increment();
            return saved;
        });
    }
}
```

The counter tracks how many orders have been created. The timer measures how long each order takes to process. Both are tagged with metadata that you can use for filtering in dashboards.

`orderTimer.record(() -> { ... })` wraps the business logic — it starts a timer before the lambda executes and stops it after, recording the duration automatically. The counter is incremented inside the lambda only after the order is saved successfully.

### Using Gauges

Gauges measure a value that fluctuates — active users, queue depth, connection pool size:

```java
@Component
public class ActiveUsersMetric {

    public ActiveUsersMetric(MeterRegistry meterRegistry,
                             SessionRegistry sessionRegistry) {
        Gauge.builder("users.active", sessionRegistry,
                registry -> registry.getAllPrincipals().size())
                .description("Number of currently active users")
                .register(meterRegistry);
    }
}
```

Unlike counters and timers, a gauge does not store values — it reads the current value from a source (the `SessionRegistry` in this case) every time it is queried.

### Using @Observed for Automatic Instrumentation

Spring Boot 4 provides the `@Observed` annotation, which automatically creates a timer and a counter for a method — no manual `MeterRegistry` code needed:

```java
@Service
public class PaymentService {

    @Observed(name = "payment.process")
    public PaymentResult processPayment(String orderId) {
        // Spring automatically tracks:
        // - payment.process.count (how many times called)
        // - payment.process.duration (how long each call takes)
        return doProcessPayment(orderId);
    }
}
```

This requires the `spring-boot-starter-aop` dependency and an `ObservedAspect` bean. `@Observed` is the simplest way to instrument a method when you do not need custom tags or conditional logic.

### Viewing Custom Metrics

Once registered, custom metrics appear at their respective endpoints:

```
GET /actuator/metrics/orders.created.total
GET /actuator/metrics/orders.processing.time
```

---

## 11.7 Securing Actuator Endpoints

Actuator endpoints can expose sensitive information — environment variables (which may contain secrets), heap dumps (which contain in-memory data), and thread dumps (which reveal application internals). In production, always secure them.

### Selective Exposure

The first line of defense is limiting which endpoints are exposed:

```properties
# Expose only safe endpoints
management.endpoints.web.exposure.include=health,info,metrics,prometheus

# Explicitly exclude sensitive endpoints
management.endpoints.web.exposure.exclude=env,beans,heapdump
```

### Securing with Spring Security

From Chapter 8, we know how to configure URL-based access rules. Apply them to Actuator paths:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health", "/actuator/info").permitAll()
            .requestMatchers("/actuator/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        );

    return http.build();
}
```

This allows public access to `/health` and `/info` (which are typically needed by load balancers and monitoring probes) while restricting all other Actuator endpoints to administrators.

### Running on a Separate Port

A common production pattern is running Actuator on a different port that is not exposed to the public internet:

```properties
# Application on port 8080 (public-facing)
server.port=8080

# Actuator on port 9090 (internal network only)
management.server.port=9090
```

The application serves user traffic on port 8080, while monitoring tools access Actuator on port 9090. The firewall blocks external access to 9090 — only internal monitoring infrastructure can reach it. This is simpler than securing endpoints with authentication and is a widely used approach.

---

## 11.8 Structured Logging

In development, human-readable log lines are fine:

```
2025-01-15 10:30:45 INFO  com.example.OrderService - Order created: id=1234
```

In production, these lines are consumed by log aggregation systems — the ELK stack (Elasticsearch, Logstash, Kibana), Grafana Loki, AWS CloudWatch, or similar tools. These systems need machine-readable formats. **Structured logging** outputs logs as JSON objects, making them easy to parse, search, and filter.

### Built-In Structured Logging

Spring Boot has built-in support for structured logging. A single property switches the console output to JSON:

```properties
logging.structured.format.console=ecs
```

Spring Boot supports three structured formats: `ecs` (Elastic Common Schema), `logstash`, and `gelf` (Graylog Extended Log Format). Choose whichever your log aggregation system expects.

### Using Logstash Logback Encoder

For more control, use the Logstash Logback Encoder library:

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

Configure it in `logback-spring.xml`:

```xml
<configuration>
    <!-- JSON logging for production -->
    <springProfile name="prod">
        <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>requestId</includeMdcKeyName>
                <includeMdcKeyName>userId</includeMdcKeyName>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON_CONSOLE"/>
        </root>
    </springProfile>

    <!-- Human-readable logging for development -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss} %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
</configuration>
```

The `<springProfile>` element from Chapter 5's profiles system selects the logging configuration based on the active profile. Development gets readable, colorful console output. Production gets structured JSON.

Example JSON log output:

```json
{
    "@timestamp": "2025-01-15T10:30:45.123Z",
    "level": "INFO",
    "logger_name": "com.example.OrderService",
    "message": "Order created successfully",
    "thread_name": "http-nio-8080-exec-1",
    "requestId": "abc-123-def",
    "userId": "user42"
}
```

### Adding Context with MDC

**MDC (Mapped Diagnostic Context)** attaches contextual data — request IDs, user IDs, session IDs — to every log message produced during a request. This lets you filter all logs for a specific request or user in your aggregation system.

```java
@Component
public class RequestIdFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        try {
            String requestId = UUID.randomUUID().toString();
            MDC.put("requestId", requestId);
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

The filter generates a unique request ID, stores it in MDC before the request is processed, and clears it afterward. Every `log.info()`, `log.error()`, or `log.debug()` call within that request automatically includes the `requestId` in the structured output. When investigating an issue, you search for the request ID and get every log message from that specific request, across all classes and layers.

---

## 11.9 Prometheus and Grafana Integration

Actuator endpoints are useful for ad-hoc inspection, but for continuous monitoring you need a system that collects metrics over time and visualizes them. **Prometheus** is a time-series database that periodically scrapes metrics from your application. **Grafana** is a visualization tool that queries Prometheus and renders dashboards.

### Adding Prometheus Support

Add the Micrometer Prometheus registry:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Expose the Prometheus endpoint:

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

This creates a `/actuator/prometheus` endpoint that outputs metrics in Prometheus text format:

```
# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap",id="G1 Eden Space"} 2.5165824E7
jvm_memory_used_bytes{area="heap",id="G1 Old Gen"} 1.8874368E7

# HELP http_server_requests_seconds Duration of HTTP server request handling
# TYPE http_server_requests_seconds summary
http_server_requests_seconds_count{method="GET",status="200",uri="/products"} 42
http_server_requests_seconds_sum{method="GET",status="200",uri="/products"} 1.234
```

Every built-in metric and every custom metric you create with Micrometer is automatically exported through this endpoint.

### Prometheus Configuration

Prometheus needs a configuration file that tells it where to scrape:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:8080']
```

Prometheus calls `/actuator/prometheus` every 15 seconds, parses the metrics, and stores them in its time-series database. You can then query this data using PromQL (Prometheus Query Language) to answer questions like "what is the average response time over the last 5 minutes?" or "how many 500 errors occurred in the last hour?"

### Grafana Dashboards

Grafana connects to Prometheus as a data source and lets you build visual dashboards. There are pre-built dashboards for Spring Boot applications available on Grafana.com (such as dashboard ID 4701) that visualize JVM memory usage, CPU usage, HTTP request rates and latencies, error rates, garbage collection activity, and your custom business metrics.

Setting up Prometheus and Grafana is typically done with Docker Compose in development:

```yaml
# docker-compose.yml (monitoring stack)
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

A detailed walkthrough of Prometheus and Grafana setup is beyond the scope of this chapter, but the integration from the Spring Boot side is simply adding the dependency and exposing the endpoint — everything else happens in the monitoring infrastructure.

---

## 11.10 Putting It All Together

Here is a complete monitoring configuration for a production application, bringing together everything from this chapter:

### application.properties

```properties
# Actuator endpoints
management.endpoints.web.exposure.include=health,info,metrics,loggers,prometheus
management.endpoint.health.show-details=when-authorized
management.info.env.enabled=true

# Application info
info.app.name=My Application
info.app.version=1.0.0
info.app.environment=production

# Structured logging (production)
logging.structured.format.console=ecs
```

### Custom Health Indicator

```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final RestClient restClient;

    public PaymentGatewayHealthIndicator(RestClient restClient) {
        this.restClient = restClient;
    }

    @Override
    public Health health() {
        try {
            restClient.get()
                    .uri("https://payments.example.com/health")
                    .retrieve()
                    .body(String.class);
            return Health.up()
                    .withDetail("gateway", "reachable")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("gateway", "unreachable")
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

### Service with Custom Metrics

```java
@Service
public class ProductService {

    private static final Logger log = LoggerFactory.getLogger(ProductService.class);
    private final ProductRepository productRepository;
    private final Counter productViewCounter;

    public ProductService(ProductRepository productRepository,
                          MeterRegistry meterRegistry) {
        this.productRepository = productRepository;
        this.productViewCounter = Counter.builder("products.views.total")
                .description("Total product page views")
                .register(meterRegistry);
    }

    public Product findById(Long id) {
        productViewCounter.increment();
        log.info("Product viewed: id={}", id);
        return productRepository.findById(id)
                .orElseThrow(() -> new ProductNotFoundException("Product not found: " + id));
    }
}
```

### Security Configuration for Actuator

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health", "/actuator/info").permitAll()
            .requestMatchers("/actuator/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        );

    return http.build();
}
```

---

## Summary

This chapter covered making a Spring Boot application observable and production-ready.

**The three pillars of observability** are logs (what happened), metrics (how much and how fast), and traces (the path a request took). Together, they give you a comprehensive view of your application's behavior in production.

**Spring Boot Actuator** adds monitoring endpoints to your application. Built-in endpoints include `/health` (application status), `/info` (metadata), `/metrics` (numeric measurements), `/loggers` (view and change log levels at runtime), and many others.

**Custom health indicators** implement the `HealthIndicator` interface to monitor external dependencies — third-party APIs, message queues, payment gateways. They appear automatically in the `/health` response.

**Micrometer** is the metrics facade. **Counters** track cumulative totals. **Gauges** measure fluctuating values. **Timers** measure both duration and count. The `@Observed` annotation in Spring Boot 4 provides automatic instrumentation without manual `MeterRegistry` code.

**Securing Actuator** involves selective exposure (only the endpoints you need), Spring Security rules (role-based access), and optionally running Actuator on a separate port that is not publicly accessible.

**Structured logging** outputs logs as JSON for machine consumption. Spring Boot's built-in `logging.structured.format.console` property enables this with a single line. **MDC** attaches contextual data (request IDs, user IDs) to every log message for easy filtering and tracing.

**Prometheus** scrapes metrics from the `/actuator/prometheus` endpoint and stores them as time series. **Grafana** visualizes those metrics as dashboards, providing real-time insight into application health, performance, and business metrics.

---

## Resources

- [Spring Boot Actuator — Reference Documentation](https://docs.spring.io/spring-boot/reference/actuator/index.html)
- [Micrometer Documentation](https://micrometer.io/docs)
- [Structured Logging — Spring Boot Reference](https://docs.spring.io/spring-boot/reference/features/logging.html#features.logging.structured)
- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Baeldung: Spring Boot Actuator](https://www.baeldung.com/spring-boot-actuators)

---

## Lab Assignment: Add Monitoring to Your Web Application

Integrate monitoring capabilities into your existing application.

**Requirements:**

1. **Add Spring Boot Actuator** and expose the `health`, `info`, `metrics`, and `loggers` endpoints.

2. **Configure the `/info` endpoint** with your application's name, version, and description — either through properties or through the Maven `build-info` goal.

3. **Create a custom health indicator** that checks the status of an external dependency your application uses (a third-party API from Chapter 10, or a database health check with a custom query).

4. **Add at least two custom metrics** using Micrometer: a **Counter** for tracking a business event (e.g., user registrations, product views, orders placed) and a **Timer** for measuring the duration of a specific operation (e.g., search query time, API call duration).

5. **Secure the Actuator endpoints** so that `/health` and `/info` are publicly accessible, while all other endpoints require the `ADMIN` role.

6. **Configure structured logging** for the production profile (JSON format) while keeping human-readable logs for the development profile. Use profile-specific configuration as covered in Chapter 5.
