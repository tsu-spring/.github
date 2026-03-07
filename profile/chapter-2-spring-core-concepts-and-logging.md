# Chapter 2: Spring Core Concepts & Logging

---

## 2.1 The ApplicationContext

In Chapter 1, we created a Spring Boot application, ran it, and watched it serve web pages. Behind that seemingly simple act, a sophisticated machinery was at work. At the center of that machinery is the **ApplicationContext** — the heart of every Spring application.

The ApplicationContext is Spring's **IoC container**. It is the object that creates, configures, wires together, and manages the lifecycle of every other object in the application. When you call `SpringApplication.run(BlogApplication.class, args)` in your `main` method, Spring Boot builds an ApplicationContext. From that moment on, the container is in control.

When the application starts, the ApplicationContext performs the following sequence:

1. It **scans** your project's packages for classes that are annotated with Spring stereotypes — `@Component`, `@Service`, `@Repository`, `@Controller`, and others.
2. It **instantiates** those classes, creating objects known as **beans**.
3. It **resolves and injects** dependencies between beans — if one bean needs another, the container provides it.
4. It makes all beans **available for use** throughout the application.

The term "bean" is simply Spring's word for an object that the container manages. A bean is not a special kind of Java object — it is an ordinary object whose creation and lifecycle happen to be controlled by Spring rather than by your code directly.

---

## 2.2 Inversion of Control

The principle that underpins the entire Spring Framework is **Inversion of Control (IoC)**.

In traditional programming, the application code is in charge. You write a `main` method, you create objects with `new`, you call methods in the order you decide. The application controls its own flow.

With Inversion of Control, this relationship is reversed. The framework takes over responsibility for creating objects and managing their interactions. Your code defines what objects exist and what they need, but the framework decides when to create them, how to wire them together, and when to destroy them.

Consider a concrete example. Suppose you have an `OrderService` that needs an `OrderRepository` to access the database. In traditional code, you might write:

```java
public class OrderService {
    private final OrderRepository orderRepository = new OrderRepository();
}
```

Here, `OrderService` is in control — it creates its own dependency. This creates a tight coupling: `OrderService` is permanently bound to that specific `OrderRepository` implementation. Testing becomes harder. Swapping implementations becomes harder. Change ripples through the codebase.

With IoC, `OrderService` does not create the repository. Instead, it declares that it needs one, and the container provides it:

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

The container sees that `OrderService` requires an `OrderRepository`, finds (or creates) a suitable bean, and passes it in. `OrderService` never calls `new OrderRepository()`. It does not know or care how the repository was created or which specific implementation it is. The control has been inverted — the container is in charge.

---

## 2.3 Dependency Injection

**Dependency Injection (DI)** is the specific mechanism through which IoC is implemented in Spring. Instead of an object creating or locating the objects it depends on, those dependencies are *injected* — handed to it from outside by the container.

Spring supports three styles of dependency injection.

### Constructor Injection (Recommended)

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

The dependency is provided through the constructor. The field can be declared `final`, guaranteeing that it is set exactly once and cannot be changed afterward. When a class has a single constructor, Spring uses it automatically — no annotation is needed.

### Setter Injection

```java
@Service
public class OrderService {
    private OrderRepository orderRepository;

    @Autowired
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

The dependency is provided through a setter method annotated with `@Autowired`. This style is appropriate for optional dependencies that may or may not be present.

### Field Injection (Not Recommended)

```java
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
}
```

The dependency is injected directly into the field. While concise, this approach has significant drawbacks: it hides dependencies from the class's public API, prevents the use of `final`, and makes unit testing difficult because you cannot easily provide test doubles through the constructor.

**Constructor injection is the recommended approach** throughout this course. It makes dependencies explicit, enforces immutability, and produces classes that are straightforward to test.

---

## 2.4 The Bean Lifecycle

Every Spring bean goes through a well-defined lifecycle, managed entirely by the ApplicationContext.

**Instantiation.** The container creates the bean object — typically by calling its constructor.

**Dependency Injection.** The container resolves and injects all dependencies that the bean requires.

**Initialization.** If the bean defines any initialization logic, it is executed at this point. The most common way to define initialization logic is with the `@PostConstruct` annotation.

**Ready for Use.** The bean is now fully initialized and available for use by other beans and by the application.

**Destruction.** When the ApplicationContext is shutting down (e.g., when the application stops), the container calls destruction callbacks. The `@PreDestroy` annotation marks a method to be called at this stage.

```java
@Component
public class CacheManager {

    @PostConstruct
    public void init() {
        // Called after all dependencies have been injected.
        // Use this for setup logic: loading caches, opening connections, etc.
    }

    @PreDestroy
    public void shutdown() {
        // Called when the application context is shutting down.
        // Use this for cleanup: closing connections, flushing buffers, etc.
    }
}
```

Understanding the lifecycle matters because it tells you where to put setup and teardown logic. Initialization code belongs in a `@PostConstruct` method — not in the constructor, because the constructor runs before dependencies are injected (in the case of setter or field injection), and not in a random business method that may or may not be called.

---

## 2.5 Bean Scopes

When the ApplicationContext creates a bean, how many instances does it create, and how long do they live? The answer depends on the bean's **scope**.

By default, every Spring bean is a **singleton** — the container creates exactly one instance, and every component that requests the bean receives the same object. For the vast majority of beans in a web application (services, repositories, controllers, configuration classes), this is the correct behavior. These objects are stateless or thread-safe, and sharing a single instance is both safe and efficient.

There are situations, however, where you need a different lifecycle:

| Scope | Behavior |
|-------|----------|
| `singleton` | One instance per ApplicationContext. This is the default. |
| `prototype` | A new instance is created every time the bean is requested. |
| `request` | One instance per HTTP request (web applications only). |
| `session` | One instance per HTTP session (web applications only). |

You declare a non-default scope with the `@Scope` annotation:

```java
@Component
@Scope("prototype")
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
    // A new ShoppingCart is created each time it is requested.
}
```

A word of caution: injecting a prototype-scoped bean into a singleton-scoped bean does not produce the behavior you might expect. The singleton is created once, so its dependencies are injected once. The prototype bean it receives will be the same instance for the entire lifetime of the singleton. If you truly need a new prototype instance on each use, you will need to look into `ObjectProvider` or method injection — topics we will revisit later in the course.

---

## 2.6 Component Scanning and Stereotype Annotations

In the examples above, we annotated classes with `@Service`, `@Component`, and `@Controller`. These are **stereotype annotations**, and they serve two purposes: they mark a class as a Spring-managed bean (so the container will detect and instantiate it), and they communicate the class's role in the application's architecture.

| Annotation | Role |
|------------|------|
| `@Component` | A generic Spring-managed bean. Use when no more specific stereotype applies. |
| `@Service` | A bean that contains business logic — the service layer. |
| `@Repository` | A bean that handles data access — the persistence layer. Spring also applies automatic exception translation to `@Repository` beans, converting database-specific exceptions into Spring's `DataAccessException` hierarchy. |
| `@Controller` | A web controller that handles HTTP requests and returns view names (templates). |
| `@RestController` | A REST controller that returns data directly (JSON, XML, etc.). It combines `@Controller` with `@ResponseBody`. |

All of these annotations are specializations of `@Component`. From the container's perspective, they all cause the class to be registered as a bean. The distinction is primarily organizational — it helps developers (and Spring itself, in some cases) understand the purpose of each class.

Spring Boot discovers these annotated classes through **component scanning**. By default, scanning begins at the package where the `@SpringBootApplication` class is located and includes all sub-packages. This is why it is important to place your main application class at the root of your package hierarchy:

```
ge.tsu.blog
├── BlogApplication.java          ← @SpringBootApplication lives here
├── controller/
│   └── PostController.java       ← found by component scanning
├── service/
│   └── PostService.java          ← found by component scanning
└── repository/
    └── PostRepository.java       ← found by component scanning
```

If a class is placed outside this package tree, it will not be found unless you explicitly configure additional scan locations.

---

## 2.7 Defining Beans Explicitly with `@Bean`

Stereotype annotations work well for classes you write yourself. But what about classes from third-party libraries? You cannot add `@Service` to a class you did not write. In these cases, you define beans explicitly using the `@Bean` annotation inside a `@Configuration` class.

```java
@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
}
```

Each method annotated with `@Bean` produces a bean. The method name becomes the bean's name by default (`restTemplate`, `objectMapper`). The return type determines the bean's type. The container calls these methods and registers the returned objects as beans in the ApplicationContext.

The `@Configuration` annotation tells Spring that this class contains bean definitions and should be processed at startup. Without it, the `@Bean` methods will be ignored.

This approach is also useful when you need custom initialization logic that goes beyond what a simple constructor can do — for example, configuring a connection pool, setting up an HTTP client with specific timeouts, or registering custom serializers.

---

## 2.8 Auto-Configuration Revisited

In Chapter 1, we mentioned that Spring Boot auto-configures your application based on the libraries present on the classpath. Now that we understand beans, the ApplicationContext, and `@Configuration` classes, we can look at this mechanism more precisely.

Auto-configuration is, at its core, a collection of `@Configuration` classes that Spring Boot ships with. Each class is responsible for configuring one particular feature. For example, there is an auto-configuration class for FreeMarker, another for the embedded Tomcat server, another for data sources, and so on.

What makes these classes special is that they are **conditional**. Each one is annotated with conditions that determine whether it should be activated:

- `@ConditionalOnClass` — activate only if a specific class is present on the classpath.
- `@ConditionalOnMissingBean` — activate only if the developer has not already defined a bean of this type.
- `@ConditionalOnProperty` — activate only if a specific configuration property is set.

This is how Spring Boot stays out of your way. If you define your own `FreeMarkerConfigurer` bean, the auto-configuration for FreeMarker detects it (via `@ConditionalOnMissingBean`) and steps aside, using your bean instead of its default.

To see exactly which auto-configurations were activated and which were skipped, add this to `application.properties`:

```properties
debug=true
```

The application will print a detailed conditions evaluation report at startup — an invaluable tool when you need to understand why something is or is not being configured.

---

## 2.9 Passing Data from Controllers to Templates

In Chapter 1, we used view controllers to map URLs directly to FreeMarker templates. View controllers are convenient for static pages, but they cannot pass data to the template. For dynamic pages — pages that display data retrieved from a service, a database, or any other source — we need a proper `@Controller` class.

Here is the basic pattern:

```java
@Controller
public class PostController {

    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }

    @GetMapping("/posts")
    public String listPosts(Model model) {
        List<Post> posts = postService.findAll();
        model.addAttribute("posts", posts);
        model.addAttribute("pageTitle", "All Posts");
        return "posts";  // resolves to templates/posts.ftlh
    }
}
```

Several things are happening here:

The class is annotated with `@Controller`, marking it as a web controller and registering it as a bean. Its dependency — `PostService` — is injected through the constructor.

The `listPosts` method is mapped to `GET /posts` via `@GetMapping`. It receives a `Model` object, which is a container for data that will be sent to the template. The method adds two attributes to the model: a list of posts and a page title. It then returns the string `"posts"`, which Spring resolves to the template file `templates/posts.ftlh`.

Inside the FreeMarker template, those model attributes become available as variables:

```html
<#import "layout.ftlh" as layout>

<@layout.page title=pageTitle>
    <h1>${pageTitle}</h1>

    <#list posts as post>
        <article>
            <h2><a href="/posts/${post.id}">${post.title}</a></h2>
            <p>${post.summary}</p>
        </article>
    </#list>
</@layout.page>
```

The `${pageTitle}` interpolation outputs the string we placed in the model. The `<#list posts as post>` directive iterates over the list. Inside the loop, `${post.title}` and `${post.summary}` access properties of each `Post` object.

This is the fundamental data flow of a Spring MVC application: the controller retrieves data, places it in the model, and names a template. FreeMarker receives the model and renders the final HTML. The controller never touches HTML; the template never touches business logic. The separation is clean and deliberate.

---

## 2.10 Introduction to Logging

Up to this point, when something went wrong or we wanted to see what our application was doing, we might have been tempted to use `System.out.println()`. That works in a trivial program, but it is entirely inadequate for a real application. Print statements cannot be turned on or off without changing code. They cannot be routed to different destinations. They carry no metadata — no timestamps, no severity levels, no class names.

**Logging** is the disciplined alternative. It is the practice of recording events that occur during an application's operation — method calls, decisions made, errors encountered, performance measurements — in a structured, configurable, and non-intrusive way.

A proper logging system gives you:

**Configurability.** You can control which messages are recorded and which are suppressed, without changing a single line of application code. During development, you might want to see every detail. In production, you might want only warnings and errors.

**Multiple Destinations.** Log messages can be written to the console, to files, to a remote logging server, or to all of these simultaneously.

**Structure.** Every log message carries metadata: a timestamp, the severity level, the class that produced it, the thread it ran on. This makes logs searchable and analyzable.

**Performance.** Modern logging frameworks are designed to have minimal overhead when a log level is disabled, so you can leave `debug` statements in your code without paying a performance penalty in production.

---

## 2.11 Default Logging in Spring Boot

The Java ecosystem has a long history of logging libraries: `java.util.logging` (built into the JDK), Apache Log4j, Logback, and several others. To avoid tying application code to a specific implementation, an abstraction layer called **SLF4J** (Simple Logging Facade for Java) was created. SLF4J defines a common API — you write your logging calls against SLF4J, and the actual logging work is done by whichever implementation is present at runtime.

Spring Boot uses this architecture out of the box:

- The **API** you program against is **SLF4J**.
- The **implementation** that does the actual work is **Logback** (which was created by the same developer who wrote SLF4J).
- By default, log messages are written to the **console**.

You do not need to add any dependencies for this — `spring-boot-starter-web` (and most other starters) include `spring-boot-starter-logging`, which brings in SLF4J and Logback automatically.

---

## 2.12 Log Levels

Every log message has a **level** that indicates its severity. The levels, in increasing order of importance, are:

| Level | Purpose |
|-------|---------|
| `TRACE` | Extremely fine-grained detail. Rarely used outside of framework internals. |
| `DEBUG` | Information useful during development — variable values, decision points, flow details. |
| `INFO` | Notable events in the application's lifecycle — startup, shutdown, key operations completed. |
| `WARN` | Something unexpected happened, but the application can continue. A potential problem that deserves attention. |
| `ERROR` | A serious failure. An operation could not be completed. Immediate attention is likely needed. |

When you set a log level, the system records messages at that level **and above**. For example, if the level is set to `INFO`, then `INFO`, `WARN`, and `ERROR` messages are recorded, while `DEBUG` and `TRACE` messages are suppressed.

Log levels are configured in `application.properties`:

```properties
# Set the root (default) level for the entire application
logging.level.root=INFO

# Set a more detailed level for your own packages
logging.level.ge.tsu.blog=DEBUG

# Suppress noisy output from a specific library
logging.level.org.hibernate.SQL=WARN
```

This granularity is one of the great strengths of a real logging system. You can see detailed output from your own code while keeping third-party library noise to a minimum.

---

## 2.13 Writing Log Messages

### The Manual Approach

To obtain a logger in any class, use SLF4J's `LoggerFactory`:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class PostService {
    private static final Logger log = LoggerFactory.getLogger(PostService.class);

    public List<Post> findAll() {
        log.info("Retrieving all posts");
        List<Post> posts = postRepository.findAll();
        log.debug("Found {} posts", posts.size());
        return posts;
    }

    public Post findById(Long id) {
        log.debug("Looking up post with id={}", id);
        return postRepository.findById(id)
            .orElseThrow(() -> {
                log.warn("Post not found: id={}", id);
                return new PostNotFoundException(id);
            });
    }
}
```

Notice the use of **parameterized messages**: `log.debug("Found {} posts", posts.size())`. The `{}` placeholder is replaced with the argument at runtime. This is important — do **not** use string concatenation (`"Found " + posts.size() + " posts"`), because the concatenation happens regardless of whether `DEBUG` is enabled. With parameterized messages, the string is constructed only if the message will actually be logged.

### The Lombok Shortcut

Since we have Lombok in our project, we can eliminate the logger declaration entirely with the `@Slf4j` annotation:

```java
@Slf4j
@Service
public class PostService {

    public List<Post> findAll() {
        log.info("Retrieving all posts");
        // ...
    }
}
```

Lombok generates the `private static final Logger log = LoggerFactory.getLogger(PostService.class)` field at compile time. The result is identical — it is purely a convenience to reduce boilerplate.

---

## 2.14 Logging to a File

Console output disappears when the application stops. For any application that runs in a server environment, you will want logs written to a file as well.

Spring Boot makes this simple:

```properties
logging.file.name=logs/app.log
```

This single property tells Logback to write log messages to a file called `app.log` inside a `logs/` directory (which will be created automatically if it does not exist). The console appender continues to work alongside the file appender — you get both.

If you need to control only the directory (and let Spring Boot choose the filename):

```properties
logging.file.path=/var/logs/blog
```

---

## 2.15 Log Patterns

Every log message is formatted according to a **pattern** — a string that defines what information appears and in what order. Spring Boot's default pattern produces output that looks something like this:

```
2025-12-01 14:30:45.123  INFO 12345 --- [main] g.t.blog.service.PostService : Retrieving all posts
```

You can customize the pattern for the console and the file independently:

```properties
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%thread] %highlight(%-5level) %cyan(%logger{36}) - %msg%n
logging.pattern.file=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
```

The most common pattern elements are:

| Element | Meaning |
|---------|---------|
| `%d{...}` | Timestamp in the specified format. |
| `%thread` | The name of the thread. |
| `%-5level` | The log level, left-padded to 5 characters. |
| `%logger{36}` | The logger name (typically the class name), truncated to 36 characters. |
| `%msg` | The log message. |
| `%n` | A newline. |
| `%highlight(...)` | Applies color coding based on log level (console only). |

---

## 2.16 Logback Configuration with `logback-spring.xml`

For requirements that go beyond what `application.properties` can express — such as multiple appenders with different levels, rolling policies, or conditional configuration based on Spring profiles — you can provide a full Logback configuration file.

Create a file named `logback-spring.xml` in `src/main/resources/`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- Console appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- File appender with rolling policy -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/archive/app-%d{yyyy-MM-dd}-%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>

    <!-- More detailed logging for our own code -->
    <logger name="ge.tsu.blog" level="DEBUG"/>
</configuration>
```

The `<include>` at the top imports Spring Boot's default Logback settings, giving you sensible baseline behavior. The `<appender>` elements define where log messages go. The `<rollingPolicy>` inside the file appender configures **log rotation**: when a log file reaches 100MB, it is archived with a date-stamped name; archives older than 30 days are deleted; and total archive size is capped at 10GB. This prevents logs from consuming unbounded disk space.

The `<root>` element sets the default level and attaches both appenders. The `<logger>` element overrides the level for a specific package.

Use `logback-spring.xml` (with the `-spring` suffix) rather than plain `logback.xml`. The Spring-aware variant supports Spring profiles and property placeholders, which plain Logback configuration does not.

---

## 2.17 Logging Best Practices

**Use SLF4J, not a specific implementation.** All your application code should import from `org.slf4j`, never from `ch.qos.logback` or `org.apache.log4j` directly. This keeps you free to change the underlying implementation without touching application code.

**Never use `System.out.println()` for logging.** It cannot be configured, filtered, or routed. Use a logger instead.

**Use parameterized messages.** Write `log.debug("User {} logged in", username)`, not `log.debug("User " + username + " logged in")`. Parameterized messages avoid unnecessary string concatenation when the level is disabled.

**Choose the right level.** `DEBUG` for information useful during development. `INFO` for significant events (startup, key operations). `WARN` for recoverable problems that deserve attention. `ERROR` for failures that require investigation.

**Be thoughtful about what you log.** A log message should be useful to someone reading it weeks later, trying to understand what happened. Include relevant context — identifiers, counts, durations — but avoid logging sensitive data such as passwords, tokens, or personal information.

**Configure log rotation.** In any environment beyond a developer's machine, configure a rolling policy to prevent log files from growing indefinitely.

---

## Summary

This chapter covered the foundational mechanisms that make Spring applications work, and introduced the logging practices that keep those applications observable.

The **ApplicationContext** is Spring's IoC container — it creates beans, injects dependencies, and manages the entire lifecycle from instantiation through destruction. **Inversion of Control** is the principle: the framework manages objects, not your code. **Dependency Injection** is the mechanism: dependencies are provided to a class from outside, typically through its constructor.

Every bean has a **lifecycle** — instantiation, injection, initialization (`@PostConstruct`), use, and destruction (`@PreDestroy`). Beans have **scopes** that determine how many instances exist; the default singleton scope is appropriate for most components.

**Stereotype annotations** (`@Component`, `@Service`, `@Repository`, `@Controller`) mark classes for component scanning and communicate architectural intent. For third-party classes or custom initialization, the `@Bean` annotation inside `@Configuration` classes provides explicit bean definitions.

**Auto-configuration** is a system of conditional `@Configuration` classes that Spring Boot evaluates at startup. It detects what is on the classpath, checks what you have already defined, and fills in the rest with sensible defaults.

When a page needs dynamic data, a **`@Controller`** retrieves it, places it in a `Model`, and returns a template name. FreeMarker receives the model attributes as variables and renders the final HTML.

**Logging** with SLF4J and Logback replaces `System.out.println()` with a configurable, structured, level-based system. Log levels (TRACE through ERROR) control what is recorded. Logs can be written to the console, to files, or both. Patterns control the format, and rolling policies prevent unbounded growth. Lombok's `@Slf4j` annotation eliminates logger boilerplate.

---

## Resources

- [Spring Framework Core — IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Spring Framework — Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies.html)
- [Spring Boot — Logging](https://docs.spring.io/spring-boot/reference/features/logging.html)
- [SLF4J Manual](https://www.slf4j.org/manual.html)
- [Logback Documentation](https://logback.qos.ch/manual/index.html)
- [Apache FreeMarker Manual](https://freemarker.apache.org/docs/index.html)
- [Spring MVC — FreeMarker Integration](https://docs.spring.io/spring-framework/reference/web/webmvc-view/mvc-freemarker.html)

---

## Lab Assignment: Add Logging to Your Blog Application

Take the blog application you built in Chapter 1 and extend it with proper logging.

**Requirements:**

1. **Refactor at least one page to use a `@Controller`** instead of a view controller. The controller should retrieve data (it can be hardcoded for now — a list of blog posts, for example), place it in the `Model`, and return a template name. The template should display the data using `${...}` and `<#list>`.

2. **Add logging to your controller and any service classes.** Use Lombok's `@Slf4j` annotation. Log at multiple levels: `INFO` for key operations (e.g., "Retrieving all posts"), `DEBUG` for details (e.g., "Found 5 posts"), and `WARN` or `ERROR` for exceptional situations.

3. **Configure log levels** in `application.properties`: set the root level to `INFO` and your application's package to `DEBUG`.

4. **Configure file logging.** Set `logging.file.name` so that logs are written to a file in addition to the console.

5. **Customize the log pattern** for either the console or the file (or both).

6. **Optional:** Create a `logback-spring.xml` configuration with a rolling file appender that rotates logs daily and keeps at most 7 days of history.
