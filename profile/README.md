# Welcome to Spring Boot Course 👋

This is the official repository for the Spring Boot course. Here you will find weekly topics, code examples, and selected student projects.

## Prerequisites

Before starting this course, students should be comfortable with:
- Java fundamentals (OOP, collections, exceptions, generics)
- Basic SQL (SELECT, INSERT, UPDATE, DELETE, JOINs)
- Basic HTML & CSS
- Version control with Git (clone, commit, push, pull)

## Recommended Tools

- **IDE:** [IntelliJ IDEA](https://www.jetbrains.com/idea/) (Community or Ultimate) or [VS Code](https://code.visualstudio.com/) with Spring Boot extensions
- **JDK:** Java 17 or later
- **Build Tool:** Maven (used in examples) or Gradle
- **Database:** H2 (embedded, for development), PostgreSQL or MySQL (for production)
- **API Testing:** [Postman](https://www.postman.com/) or `curl`
- **Containers:** [Docker Desktop](https://www.docker.com/products/docker-desktop/)

---

## Course Topics (across 14 weeks):

### [Week 1: Introduction to Spring & Spring Boot](https://github.com/tsu-spring/.github/blob/master/profile/chapter-1-introduction-to-spring-boot.md)
- What is Spring Framework and why it matters
- Spring Boot vs. traditional Spring: convention over configuration
- Setting up a project with [Spring Initializr](https://start.spring.io/)
- Understanding project structure (Maven/Gradle, `pom.xml` / `build.gradle`)
- Enabling live reload with `spring-boot-devtools`
- Serving static content (HTML, CSS, JS)
- Introduction to Thymeleaf as a template engine
- **Examples:**
  * [Static Website](https://github.com/tsu-spring/examples/tree/main/1.%20Building%20Static%20Websites/static-website)
  * [Static Website (using Bootstrap CSS)](https://github.com/tsu-spring/examples/tree/main/1.%20Building%20Static%20Websites/bootstrap-website)

### [Week 2: Spring Core Concepts & Logging](https://github.com/tsu-spring/.github/blob/master/profile/chapter-2-spring-core-concepts-and-logging.md)
- Understanding the ApplicationContext and bean lifecycle
- Inversion of Control (IoC) and Dependency Injection (DI)
- Bean scopes (`singleton`, `prototype`, `request`, `session`)
- Component scanning and stereotype annotations (`@Component`, `@Service`, `@Repository`, `@Controller`)
- Defining beans explicitly with `@Bean` in `@Configuration` classes
- Spring Boot Auto-Configuration: how it works behind the scenes
- Building dynamic pages with Thymeleaf expressions
- Introduction to logging: log levels (TRACE, DEBUG, INFO, WARN, ERROR)
- Configuring logging with SLF4J and Logback
- Writing meaningful log messages in practice
- **Examples:**
  * [Dynamic Website (Bookem All)](https://github.com/tsu-spring/examples/tree/main/2.%20Building%20Dynamic%20Websites/dynamic-website)
  * [Using Logging in Application](https://github.com/tsu-spring/examples/tree/main/9.%20Logging%20in%20Spring%20Boot/using-logging-in-application)

### Week 3: Building Interactive Web Applications
- Handling HTTP GET and POST requests
- Building and processing HTML forms with Thymeleaf
- Data binding with `@ModelAttribute`
- Server-side form validation using Bean Validation (`@Valid`, `@NotBlank`, `@Size`, etc.)
- Displaying validation errors to the user
- Handling file uploads with `MultipartFile`
- Post/Redirect/Get (PRG) pattern to prevent duplicate submissions
- Using flash attributes (`RedirectAttributes`) to pass data across redirects
- Custom error pages with Thymeleaf
- **Examples:**
  * [Interactive Web Application (Albumer)](https://github.com/tsu-spring/examples/tree/main/3.%20Building%20Interactive%20Web%20Applications/interactive-website)

### Week 4: Working with Databases
- Introduction to ORM and JPA concepts
- Defining entities with `@Entity`, `@Id`, `@GeneratedValue`
- Mapping relationships: `@OneToMany`, `@ManyToOne`, `@ManyToMany`
- Creating repositories with Spring Data JPA (`JpaRepository`, `CrudRepository`)
- Derived query methods and `@Query` (JPQL)
- Pagination and sorting with `Pageable` and `Page<T>`
- Building paginated views with Thymeleaf
- Transaction management basics with `@Transactional`
- Using H2 for development and PostgreSQL/MySQL for production
- Database initialization with `data.sql` and `schema.sql`
- Brief intro to schema migration tools (Flyway, Liquibase) for production use
- **Examples:**
  * [Working with Databases (Gossip Nation)](https://github.com/tsu-spring/examples/tree/main/4.%20Working%20with%20Databases/website-with-database)

### Week 5: Configuration & Profiles
- Working with `application.properties` and `application.yml`
- Binding configuration to Java objects with `@ConfigurationProperties`
- Using `@Value` for simple property injection
- Spring profiles: separating `dev`, `test`, and `prod` environments
- Profile-specific configuration files (`application-dev.yml`, etc.)
- Externalized configuration: environment variables, command-line arguments
- Practical use case: switching between H2 and PostgreSQL with profiles
- **Examples:**
  * [Custom Configurations and Profiles](https://github.com/tsu-spring/examples/tree/main/8.%20Custom%20Configurations%20and%20Profiles/custom-configurations-and-profiles)

### Week 6: Building RESTful Web Services
- REST principles and HTTP methods (GET, POST, PUT, PATCH, DELETE)
- Building REST controllers with `@RestController` and `@RequestMapping`
- Path variables, query parameters, and request bodies
- Using DTOs to shape API responses
- Controlling JSON serialization (`@JsonIgnore`, `@JsonProperty`, handling JPA relationship recursion)
- Global exception handling with `@ControllerAdvice` and `@ExceptionHandler`
- HTTP status codes and `ResponseEntity`
- Introduction to API documentation with Swagger/OpenAPI
- Testing APIs manually with Postman or curl
- **Examples:**
  * [Building Web Services (Fake IMDB REST API)](https://github.com/tsu-spring/examples/tree/main/5.%20Building%20Web%20Services/restfull-web-app)

### Week 7: Testing Web Applications
- Why testing matters: the testing pyramid (unit, integration, end-to-end)
- Unit testing with JUnit 5 and Mockito
- Writing testable code: constructor injection and loose coupling
- Integration testing with `@SpringBootTest`
- Testing slices: `@WebMvcTest`, `@DataJpaTest`
- Testing REST APIs with `MockMvc`
- Using test profiles and embedded databases (H2) for test isolation
- **Examples:**
  * [Application with Tests](https://github.com/tsu-spring/examples/tree/main/10.%20Testing%20Web%20Applications/application-with-tests)

### Week 8: Securing Web Applications
- Why security matters: common vulnerabilities (CSRF, XSS, SQL injection)
- Adding Spring Security to a project
- Authentication: in-memory, JDBC-based, and custom `UserDetailsService`
- Password encoding with `BCryptPasswordEncoder`
- Authorization: role-based access control (`@PreAuthorize`, `hasRole()`)
- Securing web pages and REST endpoints
- Customizing login/logout pages
- CORS configuration for REST APIs
- Brief overview of token-based authentication (JWT)
- **Examples:**
  * [Securing Web Applications (Blogium)](https://github.com/tsu-spring/examples/tree/main/6.%20Securing%20Web%20Applications/secured-web-app)

### Week 9: Internationalization & Localization
- Understanding i18n and l10n concepts
- Setting up message bundles (`messages.properties`, `messages_ka.properties`)
- Using `MessageSource` and Thymeleaf's `#{...}` syntax
- Locale resolution strategies (`AcceptHeaderLocaleResolver`, `SessionLocaleResolver`)
- Switching languages with `LocaleChangeInterceptor`
- Formatting dates, numbers, and currencies by locale
- **Examples:**
  * [Multilingual Website](https://github.com/tsu-spring/examples/tree/main/7.%20Multilingual%20Websites/multilingual-website)

### Week 10: Consuming External APIs & Caching
- Calling external REST APIs with `RestClient` (Spring Boot 3.2+)
- Handling API responses and error scenarios
- Mapping JSON responses to Java objects
- Introduction to caching with `@Cacheable`, `@CacheEvict`, and `@EnableCaching`
- When and where to use caching effectively
- Brief overview of cache providers (Caffeine, Redis)

### Week 11: Monitoring Web Applications
- Introduction to observability: why monitoring matters in production
- Adding and configuring Spring Boot Actuator
- Built-in endpoints: `/health`, `/info`, `/metrics`, `/env`
- Creating custom health indicators
- Custom metrics with Micrometer
- Exposing and securing Actuator endpoints
- Structured logging (JSON format) for production
- Overview of integrations with Prometheus and Grafana
- **Examples:**
  * [Application with Monitoring](https://github.com/tsu-spring/examples/tree/main/11.%20Monitoring%20Web%20Applications/application-with-monitoring)

### Week 12: Messaging & Asynchronous Processing
- Synchronous vs. asynchronous communication patterns
- Introduction to message brokers and JMS
- Setting up ActiveMQ with Spring Boot
- Sending and receiving messages with `JmsTemplate` and `@JmsListener`
- Asynchronous method execution with `@Async` and `@EnableAsync`
- Scheduling tasks with `@Scheduled` and `@EnableScheduling`
- Sending emails with `JavaMailSender` and Thymeleaf templates
- Introduction to WebSockets: real-time communication with STOMP and SockJS
- **Examples:**
  * [JMS ActiveMQ 5 (Classic) Demo](https://github.com/tsu-spring/examples/tree/main/12.%20Messaging%20and%20Asynchronous%20Processing/jms-activemq-demo)

### Week 13: Microservices Architecture with Spring Boot
- Monolith vs. microservices: trade-offs and when to use each
- Decomposing a monolith into services
- Inter-service communication with `RestClient` and messaging
- Overview of Spring Cloud ecosystem:
  - Service discovery (Eureka)
  - API Gateway (Spring Cloud Gateway)
  - Centralized configuration (Spring Cloud Config)
  - Circuit breakers and resilience (Resilience4j)
- _Note: This week focuses on concepts and a live demo rather than a full hands-on project._

### Week 14: Deploying Spring Boot Applications
- Packaging as JAR vs. WAR: differences and use cases
- Building executable JARs with Maven/Gradle
- Introduction to Docker: writing a `Dockerfile` for Spring Boot
- Docker Compose for multi-container setups (app + database)
- Overview of cloud deployment options (Railway, Render, fly.io)
- Environment-specific configuration in production
- Brief intro to CI/CD with GitHub Actions (build & test pipeline)
- **Examples:**
  * [Traditional WAR Deployment](https://github.com/tsu-spring/examples/tree/main/14.%20Deploying%20Web%20Applications/traditional-deployment)

---

## Student Projects (Selected)
### 2025 (1)
* [Bored Reader](https://github.com/tsu-spring/boredreader) - A website for reading books and mangas, where you can chat with AI assistants.

---

## Spring Boot References

### 📘 Official & Core Guides

Always up-to-date and maintained by the Spring team.

🔗 **Spring Boot Official Tutorials (Spring.io)** – hands-on guided tutorials straight from Spring. ([Home][2])
[https://docs.spring.io/spring-boot/tutorial/](https://docs.spring.io/spring-boot/tutorial/)

🔗 **Spring Boot Reference Documentation** – full official detailed documentation. ([BSWEN][3])
[https://docs.spring.io/spring-boot/docs/current/reference/html/](https://docs.spring.io/spring-boot/docs/current/reference/html/)

### 🧠 In-Depth Written Tutorials 

Great for learning concepts, annotations, dependency injection, REST APIs, etc.

🔗 **Baeldung Spring Boot Series** — high-quality articles with examples. ([Baeldung on Kotlin][1])
[https://www.baeldung.com/spring-boot](https://www.baeldung.com/spring-boot)

🔗 **Spring Boot Tutorials (GeeksforGeeks)** — beginner-friendly step-by-step explanations. ([GeeksforGeeks][4])
[https://www.geeksforgeeks.org/spring-boot/](https://www.geeksforgeeks.org/spring-boot/)

🔗 **Spring Boot Handbook — Learn Code With Durgesh** — free tutorials for beginners. ([learncodewithdurgesh.com][5])
[https://learncodewithdurgesh.com/tutorials/spring-boot-tutorials](https://learncodewithdurgesh.com/tutorials/spring-boot-tutorials)

### 🎥 Video Tutorials & Courses

Excellent for visual learners — follow along with code examples:

🔗 **Spring Boot Full Course — Amigoscode (YouTube)** — walkthrough from basics to advanced. ([Glasp][6])
[https://www.youtube.com/watch?v=9SGDpanrc8U](https://www.youtube.com/watch?v=9SGDpanrc8U)

🔗 **Spring Boot Project for Beginners — Telusko (YouTube)** — practical beginner project. ([YouTube][7])
[https://www.youtube.com/watch?v=vlz9ina4Usk](https://www.youtube.com/watch?v=vlz9ina4Usk)

### 📚 Courses & Structured Learning

Free or audit-friendly structured courses you can follow:

🔗 **Spring Boot Course on Coursera (audit available)** — covers fundamentals and common patterns. ([BSWEN][3])
[https://www.coursera.org/learn/spring-boot](https://www.coursera.org/learn/spring-boot)

🔗 **Spring Boot Developer Track — Hyperskill** — interactive projects and exercises. ([BSWEN][3])
[https://hyperskill.org/tracks](https://hyperskill.org/tracks)

### 📝 Extras & Community Curated Resources

For additional reading, project ideas, examples, and community textbooks:

🔗 **Awesome Spring / Spring Boot Resources (GitHub)** — curated list with links to docs, courses, books, examples etc. ([GitHub][8])
[https://github.com/spring-office-hours/resources-learning-spring](https://github.com/spring-office-hours/resources-learning-spring)
