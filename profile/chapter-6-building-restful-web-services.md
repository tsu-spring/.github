# Chapter 6: Building RESTful Web Services

---

## 6.1 From Server-Rendered Pages to APIs

In the previous chapters, we built a web application that renders HTML on the server — controllers process requests, populate a model, and return a FreeMarker template that the browser displays. This works well when the server controls the entire user experience. But modern applications often need to serve data to clients that are not browsers — mobile apps, single-page JavaScript applications, other backend services, IoT devices. These clients do not want HTML. They want structured data, typically JSON, that they can consume and present however they choose.

This chapter introduces RESTful web services — HTTP-based APIs that expose your application's data and operations to any client that speaks HTTP. We will learn the principles behind REST, how to build REST controllers with Spring Boot, how to handle different types of request input (path variables, query parameters, request bodies), how to shape API responses with Data Transfer Objects (DTOs), how to control JSON serialization with Jackson, how to provide meaningful HTTP status codes with `ResponseEntity`, how to handle errors consistently across all endpoints, and how to document your API with SpringDoc OpenAPI.

---

## 6.2 What Are Web Services?

A web service is a software system that exposes functionality over a network using standard protocols. In practical terms, it is a set of endpoints that another program can call — not by clicking a link in a browser, but by sending an HTTP request programmatically.

Two architectural styles have dominated web services over the years: **SOAP** and **REST**.

### SOAP (Brief Overview)

SOAP (Simple Object Access Protocol) is an XML-based protocol with strict standards defined by the W3C. It uses XML for all messages, supports multiple transport protocols (HTTP, SMTP, TCP), and relies on a formal contract (WSDL) to describe the service interface. SOAP is powerful in enterprise contexts that require transactional guarantees, formal contracts, and protocol-level security (WS-Security). However, its verbosity, complexity, and rigid structure make it a poor fit for most modern web and mobile applications.

### REST

REST (Representational State Transfer) is an architectural style — not a protocol — defined by a set of constraints that, when followed, produce services that are simple, scalable, and aligned with how HTTP already works. REST has become the dominant approach for building web APIs, and it is what we will use exclusively in this chapter.

---

## 6.3 REST Principles

A RESTful service follows several key principles:

**Resources identified by URIs.** Every piece of data your API exposes — a user, a product, a blog post — is a resource, and every resource has a unique URI: `/users/42`, `/products/7`, `/posts/15/comments`.

**Standard HTTP methods.** Instead of inventing custom operations, REST uses the HTTP methods that already exist: GET to retrieve, POST to create, PUT to replace, PATCH to partially update, DELETE to remove.

**Stateless communication.** Each request contains all the information the server needs to process it. The server does not remember anything about previous requests. This makes the service easier to scale — any server instance can handle any request.

**Client-server separation.** The client (a browser, a mobile app, another service) and the server are independent. The server exposes an API; the client consumes it. Neither depends on the other's implementation details.

**JSON as the data format.** While REST does not mandate a specific format, JSON has become the universal standard for REST APIs. It is lightweight, human-readable, and natively supported by every modern programming language.

A client interacts with a REST API using standard HTTP tools. From the command line:

```bash
curl -X GET http://api.example.com/users/1
```

From JavaScript:

```javascript
fetch('http://api.example.com/users/1')
  .then(response => response.json())
  .then(data => console.log(data));
```

The same API serves both. That is the point.

---

## 6.4 HTTP Methods and Their Semantics

Each HTTP method carries a specific meaning in a REST API:

| Method | Purpose | Typical Usage |
|--------|---------|---------------|
| **GET** | Retrieve a resource or collection | `GET /users` — list all users; `GET /users/1` — get user 1 |
| **POST** | Create a new resource | `POST /users` — create a new user |
| **PUT** | Replace a resource entirely | `PUT /users/1` — replace user 1 with the provided data |
| **PATCH** | Partially update a resource | `PATCH /users/1` — update specific fields of user 1 |
| **DELETE** | Remove a resource | `DELETE /users/1` — delete user 1 |

GET and DELETE typically have no request body. POST, PUT, and PATCH send data in the request body (usually as JSON). GET is safe (it never modifies data) and idempotent (calling it multiple times produces the same result). PUT and DELETE are idempotent. POST is neither.

---

## 6.5 Designing RESTful URIs

Good URI design makes an API intuitive and predictable. A few conventions that the industry has settled on:

**Use nouns, not verbs.** The URI identifies a resource — what it *is*, not what you *do* with it. The HTTP method provides the action. `/users` is correct; `/getUsers` is not.

**Use plural names for collections.** `/users`, `/posts`, `/products` — not `/user` or `/post`.

**Use hierarchical paths for nested resources.** If comments belong to a post, the URI reflects that relationship: `/posts/{postId}/comments`.

**Use lowercase and hyphens.** If a resource name contains multiple words, use hyphens: `/blog-posts`, not `/blogPosts` or `/blog_posts`.

Here is a complete URI design for a blog application:

```
GET    /posts                              — list all posts
GET    /posts/{postId}                     — get a specific post
POST   /posts                              — create a new post
PUT    /posts/{postId}                     — update a post
DELETE /posts/{postId}                     — delete a post

GET    /posts/{postId}/comments            — list comments on a post
GET    /posts/{postId}/comments/{commentId} — get a specific comment
POST   /posts/{postId}/comments            — add a comment to a post
PUT    /posts/{postId}/comments/{commentId} — update a comment
DELETE /posts/{postId}/comments/{commentId} — delete a comment
```

The structure is consistent and self-describing. A developer seeing this API for the first time can predict how it works.

---

## 6.6 Building REST Controllers with @RestController

In the previous chapters, our controllers used `@Controller` and returned template names. For a REST API, we use `@RestController` instead. The difference is simple but fundamental: `@RestController` combines `@Controller` and `@ResponseBody`, which means every handler method's return value is serialized directly into the HTTP response body (as JSON, by default) rather than being interpreted as a view name.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }

    @PostMapping
    public User createUser(@RequestBody User newUser) {
        return userService.save(newUser);
    }
}
```

When a client sends `GET /api/users`, Spring calls `getAllUsers()`, which returns a `List<User>`. Because the class is annotated with `@RestController`, Spring serializes that list to JSON and writes it to the response body. The client receives something like:

```json
[
  { "id": 1, "name": "Alice", "email": "alice@example.com" },
  { "id": 2, "name": "Bob", "email": "bob@example.com" }
]
```

No template. No model. Just data.

---

## 6.7 Path Variables, Query Parameters, and Request Bodies

REST controllers accept input in three ways, each suited to a different purpose.

### Path Variables

Path variables extract values from the URI path itself. They identify a specific resource:

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

A request to `GET /users/42` passes `42` as the `id` parameter. Spring handles the type conversion from the string in the URL to a `Long`.

### Query Parameters

Query parameters are appended to the URL after a `?` and are typically used for filtering, searching, pagination, and sorting — operations that refine a collection rather than identify a specific resource:

```java
@GetMapping("/users")
public List<User> searchUsers(
        @RequestParam(required = false) String name,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {
    return userService.search(name, page, size);
}
```

A request to `GET /users?name=Alice&page=0&size=20` passes `"Alice"` as `name`, `0` as `page`, and `20` as `size`. The `required = false` attribute makes `name` optional — if omitted, it will be `null`. The `defaultValue` attribute provides a fallback when the parameter is not present.

### Request Bodies

POST, PUT, and PATCH requests typically send data in the request body as JSON. The `@RequestBody` annotation tells Spring to deserialize the JSON into a Java object:

```java
@PostMapping("/users")
public User createUser(@RequestBody User newUser) {
    return userService.save(newUser);
}
```

When the client sends:

```json
{ "name": "Charlie", "email": "charlie@example.com" }
```

Spring deserializes it into a `User` object, populating the `name` and `email` fields. The `@RequestBody` annotation triggers Jackson (Spring Boot's JSON library) to perform the conversion.

---

## 6.8 Using DTOs to Shape API Responses

In the examples above, we returned entity objects directly. This works for simple cases but becomes a problem quickly. Your JPA entities may contain fields that should not be exposed (passwords, internal flags), bidirectional relationships that cause infinite recursion during serialization, or implementation details that you do not want to become part of your API contract. When you return an entity directly, your database schema *is* your API — and any change to the schema breaks every client.

**Data Transfer Objects (DTOs)** solve this by providing a separate class that represents exactly what the API should return — nothing more, nothing less.

### Defining a DTO

```java
public class UserDTO {

    private Long id;
    private String name;
    private String email;

    public UserDTO(User user) {
        this.id = user.getId();
        this.name = user.getName();
        this.email = user.getEmail();
    }

    // getters and setters
}
```

The DTO contains only the fields the client should see. The `User` entity might have a `password` field, a `role` field, and a bidirectional relationship to `Post` — none of that appears in the DTO.

### Using DTOs in the Controller

```java
@GetMapping("/users/{id}")
public UserDTO getUser(@PathVariable Long id) {
    User user = userService.findById(id);
    return new UserDTO(user);
}

@GetMapping("/users")
public List<UserDTO> getAllUsers() {
    return userService.findAll().stream()
            .map(UserDTO::new)
            .toList();
}
```

The pattern is simple: the service returns entities, and the controller (or a dedicated mapper) converts them to DTOs before returning them. This keeps the API contract stable even when the entity changes.

### Request DTOs

The same principle applies to input. Instead of accepting a full `User` entity in a POST request, define a `CreateUserRequest` that contains only the fields the client should provide:

```java
public class CreateUserRequest {

    @NotBlank
    private String name;

    @Email
    @NotBlank
    private String email;

    // getters and setters
}
```

```java
@PostMapping("/users")
public ResponseEntity<UserDTO> createUser(@Valid @RequestBody CreateUserRequest request) {
    User user = userService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(new UserDTO(user));
}
```

This separates what the client sends from what the server stores and what the server returns — three potentially different shapes of data, each with its own class.

---

## 6.9 Controlling JSON Serialization with Jackson

Spring Boot uses **Jackson** to convert Java objects to JSON and back. In Spring Boot 4, Jackson 3 is the default JSON library. It is auto-configured — when `spring-boot-starter-web` is on the classpath, Spring Boot registers a `JsonMapper` bean that handles serialization and deserialization automatically.

Jackson 3 introduces several changes from Jackson 2. The primary entry point is now `JsonMapper` (an immutable builder-based mapper) rather than the mutable `ObjectMapper` of Jackson 2. The package has moved from `com.fasterxml.jackson` to `tools.jackson` — with one important exception: annotations like `@JsonProperty`, `@JsonIgnore`, and `@JsonFormat` still live in the `com.fasterxml.jackson.annotation` package for backward compatibility. Dates and times are serialized as ISO-8601 strings by default (e.g., `"2025-03-17T09:30:00"` instead of a numeric timestamp), which is the behavior most APIs want.

### Global Configuration

You can customize Jackson's behavior in `application.properties`:

```properties
spring.jackson.default-property-inclusion=non_null
```

This tells Jackson to omit fields that are `null` from the JSON output, keeping responses compact.

### Key Jackson Annotations

When global configuration is not enough, Jackson provides annotations that control serialization at the field or class level:

| Annotation | Purpose |
|------------|---------|
| `@JsonProperty("display_name")` | Use a different name in JSON than the Java field name. |
| `@JsonIgnore` | Exclude a field from serialization and deserialization entirely. |
| `@JsonIgnoreProperties({"field1", "field2"})` | Ignore specified properties at the class level. |
| `@JsonInclude(JsonInclude.Include.NON_NULL)` | Omit the field from JSON when it is null. |
| `@JsonFormat(pattern = "yyyy-MM-dd")` | Control date/time formatting. |
| `@JsonManagedReference` / `@JsonBackReference` | Handle bidirectional relationships (break infinite recursion). |

### Handling Bidirectional JPA Relationships

If you return JPA entities directly (which you generally should not — use DTOs), bidirectional relationships will cause infinite recursion: serializing a `Post` includes its `Comment` list, each `Comment` includes its `Post`, which includes its `Comment` list, and so on forever.

There are several solutions:

**Option 1: `@JsonManagedReference` and `@JsonBackReference`**

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post")
    @JsonManagedReference
    private List<Comment> comments;
}

@Entity
public class Comment {
    @ManyToOne
    @JsonBackReference
    private Post post;
}
```

The "managed" side is serialized normally. The "back" side is excluded during serialization. This breaks the cycle but means you can never see the parent from the child's JSON.

**Option 2: `@JsonIgnore`**

```java
@Entity
public class Comment {
    @ManyToOne
    @JsonIgnore
    private Post post;
}
```

Simpler, but the field is completely invisible in JSON.

**Option 3: Use DTOs (recommended).** Map entities to DTOs that do not contain circular references. This is the cleanest solution — it separates your serialization concerns from your persistence model entirely.

---

## 6.10 Controlling Responses with ResponseEntity

Returning a plain Java object from a controller method works, but it always produces a 200 OK response. REST APIs need to communicate more precisely: 201 Created when a resource is created, 204 No Content when a deletion succeeds, 404 Not Found when a resource does not exist. `ResponseEntity` gives you full control over the HTTP response — status code, headers, and body.

```java
@GetMapping("/users/{id}")
public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
    return userService.findById(id)
            .map(user -> ResponseEntity.ok(new UserDTO(user)))
            .orElse(ResponseEntity.notFound().build());
}

@PostMapping("/users")
public ResponseEntity<UserDTO> createUser(@Valid @RequestBody CreateUserRequest request) {
    User saved = userService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(new UserDTO(saved));
}

@DeleteMapping("/users/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
}
```

`ResponseEntity.ok(body)` returns 200 with the given body. `ResponseEntity.status(HttpStatus.CREATED).body(body)` returns 201. `ResponseEntity.notFound().build()` returns 404 with no body. `ResponseEntity.noContent().build()` returns 204 with no body.

You can also add custom headers:

```java
@GetMapping("/example")
public ResponseEntity<String> example() {
    HttpHeaders headers = new HttpHeaders();
    headers.add("X-Custom-Header", "CustomValue");
    return new ResponseEntity<>("Response body", headers, HttpStatus.OK);
}
```

Using `ResponseEntity` consistently is a best practice. It makes your API's behavior explicit in the code — you can see exactly what status code each endpoint returns without guessing.

---

## 6.11 Error Handling

When something goes wrong — a resource is not found, validation fails, an unexpected exception occurs — your API needs to return a meaningful error response instead of a generic 500 or a stack trace. Clients need structured error information they can parse and act on.

### Spring Boot's Default Error Response

Spring Boot provides built-in error handling. When an exception is thrown, it returns a JSON error response:

```json
{
    "timestamp": "2025-03-17T09:17:47.837+00:00",
    "status": 500,
    "error": "Internal Server Error",
    "path": "/api/users/999"
}
```

You can control what appears in this response through properties:

```properties
server.error.include-message=never
server.error.include-exception=false
server.error.include-stacktrace=never
```

In production, keep all of these disabled — they can leak implementation details. During development, you can enable `include-message` and `include-stacktrace` for debugging.

### Global Exception Handling with @RestControllerAdvice

The default error response is limited and inconsistent with the shape of your normal API responses. For a production API, you want centralized exception handling that returns structured error objects in a consistent format.

The `@RestControllerAdvice` annotation (a combination of `@ControllerAdvice` and `@ResponseBody`) creates a global exception handler that intercepts exceptions from all `@RestController` classes:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(HttpStatus.NOT_FOUND.value(), ex.getMessage());
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        ErrorResponse error = new ErrorResponse(HttpStatus.BAD_REQUEST.value(), message);
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneralException(Exception ex) {
        ErrorResponse error = new ErrorResponse(
                HttpStatus.INTERNAL_SERVER_ERROR.value(),
                "An unexpected error occurred.");
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

### The Error Response Class

```java
public class ErrorResponse {

    private int status;
    private String message;
    private LocalDateTime timestamp;

    public ErrorResponse(int status, String message) {
        this.status = status;
        this.message = message;
        this.timestamp = LocalDateTime.now();
    }

    // getters and setters
}
```

Now every error — whether it is a missing resource, a validation failure, or an unexpected exception — returns a consistent JSON structure:

```json
{
    "status": 404,
    "message": "User not found with id: 42",
    "timestamp": "2025-03-17T09:30:00"
}
```

The `@ExceptionHandler(ResourceNotFoundException.class)` method catches any `ResourceNotFoundException` thrown by any controller in the application. The `@ExceptionHandler(MethodArgumentNotValidException.class)` method catches validation errors triggered by `@Valid` on `@RequestBody` parameters — it extracts the field-level errors and formats them into a readable message. The catch-all `@ExceptionHandler(Exception.class)` handles anything unexpected, returning a generic message rather than leaking internal details.

### Defining a Custom Exception

The `ResourceNotFoundException` is a simple custom exception:

```java
public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

Used in the service layer:

```java
public User findById(Long id) {
    return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found with id: " + id));
}
```

This pattern — throw an exception in the service, catch it in the global handler, return a structured response — keeps error handling clean and consistent across the entire API.

### A Note on @ControllerAdvice

`@ControllerAdvice` (and its `@RestControllerAdvice` variant) is not limited to exception handling. It can also define shared model attributes across all controllers using `@ModelAttribute` methods:

```java
@ControllerAdvice
public class SharedModelAdvice {

    @ModelAttribute("appName")
    public String appName() {
        return "My Application";
    }
}
```

This is useful in server-rendered applications (with `@Controller` and templates) — every template gets access to `appName` without each controller adding it manually. For REST APIs, this is rarely needed, but it is worth knowing that `@ControllerAdvice` is a general-purpose mechanism, not just an error handler.

---

## 6.12 API Documentation with SpringDoc OpenAPI

An API that is not documented is an API that no one can use — or rather, one that everyone uses incorrectly. **SpringDoc OpenAPI** generates interactive API documentation automatically from your Spring Boot code.

### Setup

Add the dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>3.0.0</version>
</dependency>
```

That is all. Start your application, and two endpoints become available:

- **OpenAPI specification** (JSON): `http://localhost:8080/v3/api-docs`
- **Swagger UI** (interactive documentation): `http://localhost:8080/swagger-ui.html`

Swagger UI provides a web interface where you can see every endpoint in your API, inspect the expected request and response formats, and send test requests directly from the browser.

SpringDoc reads your `@RestController` classes, their `@GetMapping` / `@PostMapping` annotations, `@PathVariable` and `@RequestParam` parameters, `@RequestBody` types, and return types to generate the documentation. The more expressive your code is (using descriptive parameter names, DTOs with clear field names, meaningful response types), the better the auto-generated documentation will be.

### Disabling in Production

In production, you probably do not want API documentation endpoints exposed. Disable them with a property:

```properties
springdoc.api-docs.enabled=false
springdoc.swagger-ui.enabled=false
```

A natural place for this is in your `application-prod.properties` — documentation available in development, hidden in production.

---

## 6.13 Testing APIs with curl and Postman

Before your API reaches any client application, you should test it manually to verify that endpoints return the correct status codes, response bodies, and error handling. Two tools are commonly used.

### curl

`curl` is a command-line tool available on every major operating system. It is fast, scriptable, and requires no installation:

```bash
# GET all users
curl -X GET http://localhost:8080/api/users

# GET a specific user
curl -X GET http://localhost:8080/api/users/1

# POST — create a new user
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com"}'

# PUT — update a user
curl -X PUT http://localhost:8080/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice Updated", "email": "alice@example.com"}'

# DELETE — remove a user
curl -X DELETE http://localhost:8080/api/users/1
```

The `-H "Content-Type: application/json"` header tells the server that the request body is JSON. The `-d` flag provides the body content. Add `-v` for verbose output that shows request and response headers, or pipe the output to `jq` for pretty-printed JSON.

### Postman

Postman is a graphical tool for building, testing, and documenting API requests. It is especially useful when you need to manage collections of requests, set up authentication headers, inspect response bodies visually, or share test suites with your team. The workflow is straightforward: select the HTTP method, enter the URL, set headers and body content (for POST/PUT), and click Send.

Both tools accomplish the same thing. Use whichever fits your workflow — `curl` for quick checks from the terminal, Postman for more involved testing sessions.

---

## 6.14 Putting It All Together: A Complete REST Controller

Here is a complete controller that combines everything covered in this chapter — DTOs, validation, `ResponseEntity`, and proper HTTP status codes:

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public List<UserDTO> getAllUsers() {
        return userService.findAll().stream()
                .map(UserDTO::new)
                .toList();
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        return userService.findById(id)
                .map(user -> ResponseEntity.ok(new UserDTO(user)))
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<UserDTO> createUser(@Valid @RequestBody CreateUserRequest request) {
        User saved = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(new UserDTO(saved));
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserDTO> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        User updated = userService.update(id, request);
        return ResponseEntity.ok(new UserDTO(updated));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

Each method returns an appropriate HTTP status code. Input is validated with `@Valid`. Entities are never exposed directly — DTOs control what the client sees. The controller is thin: it delegates business logic to the service and focuses on HTTP concerns.

---

## 6.15 Beyond the Basics

Two libraries are worth mentioning for when your API needs grow beyond what this chapter covers.

### Spring HATEOAS

HATEOAS (Hypermedia as the Engine of Application State) is a REST principle that says API responses should include links telling the client what it can do next. Instead of the client hardcoding URLs, the API provides navigational information alongside the data:

```json
{
    "id": 1,
    "name": "John Doe",
    "role": "Developer",
    "_links": {
        "self": { "href": "http://localhost:8080/employees/1" },
        "employees": { "href": "http://localhost:8080/employees" }
    }
}
```

Spring HATEOAS provides the model classes and utilities to build these link-enriched responses.

### Spring Data REST

Spring Data REST takes a different approach entirely — it automatically exposes your Spring Data repositories as RESTful endpoints with no controller code at all:

```java
@RepositoryRestResource(path = "products")
public interface ProductRepository extends JpaRepository<Product, Long> {
}
```

This single interface generates `GET /products`, `GET /products/{id}`, `POST /products`, `PUT /products/{id}`, and `DELETE /products/{id}` — complete with pagination, sorting, and HATEOAS links. It is powerful for prototyping or for APIs that closely mirror your data model, but it offers less control than hand-written controllers.

---

## Summary

This chapter covered the tools and patterns for building RESTful web services with Spring Boot.

**REST** is an architectural style where resources are identified by URIs, operations use standard HTTP methods, communication is stateless, and data is exchanged as JSON.

**`@RestController`** creates controllers where every method's return value is serialized to JSON. It replaces `@Controller` when you are building an API rather than rendering templates.

**Path variables** (`@PathVariable`) identify specific resources. **Query parameters** (`@RequestParam`) filter, search, and paginate collections. **Request bodies** (`@RequestBody`) carry JSON data for create and update operations.

**DTOs** decouple your API contract from your database entities, preventing sensitive data leaks, infinite recursion, and tight coupling between your schema and your clients.

**Jackson** handles JSON serialization and deserialization. In Spring Boot 4, Jackson 3 is the default, with `JsonMapper` as the primary entry point. Annotations like `@JsonIgnore`, `@JsonProperty`, and `@JsonFormat` provide fine-grained control. Dates are serialized as ISO-8601 strings by default.

**`ResponseEntity`** gives explicit control over HTTP status codes, headers, and the response body — essential for a well-behaved REST API.

**`@RestControllerAdvice`** with `@ExceptionHandler` provides centralized error handling, returning consistent, structured error responses across all endpoints.

**SpringDoc OpenAPI** generates interactive API documentation automatically from your code, available through Swagger UI during development.

---

## Resources

- [Building a RESTful Web Service — Spring Guide](https://spring.io/guides/gs/rest-service/)
- [Building REST Services with Spring — Tutorial](https://spring.io/guides/tutorials/rest/)
- [JSON Support — Spring Boot](https://docs.spring.io/spring-boot/reference/features/json.html)
- [Controller Advice — Spring Framework](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-advice.html)
- [Error Handling for REST with Spring — Baeldung](https://www.baeldung.com/exception-handling-for-rest-with-spring)
- [SpringDoc OpenAPI](https://springdoc.org/)
- [Spring HATEOAS](https://spring.io/projects/spring-hateoas)
- [Spring Data REST](https://spring.io/projects/spring-data-rest)

---

## Lab Assignment: Build a RESTful API

Add REST web services to your existing web application.

**Requirements:**

1. **Create a REST controller** using `@RestController` with a base path like `/api/...`. Implement endpoints for all CRUD operations (GET, POST, PUT, DELETE) on at least one of your entities.

2. **Use DTOs** for all API responses — do not expose JPA entities directly. Define separate request DTOs (with validation annotations) for create and update operations.

3. **Implement path variables and query parameters.** Use `@PathVariable` for resource identification and `@RequestParam` for filtering or searching (e.g., search posts by title, filter by category).

4. **Return appropriate HTTP status codes** using `ResponseEntity`: 200 for successful retrieval, 201 for creation, 204 for deletion, 404 for not found.

5. **Implement centralized exception handling** with `@RestControllerAdvice`. Define at least a `ResourceNotFoundException` handler and a validation error handler. Return a consistent error response structure across all endpoints.

6. **Use Jackson annotations** where appropriate (e.g., `@JsonIgnore` to hide fields, `@JsonFormat` to format dates, `@JsonProperty` to rename fields).

7. **Document your API** using SpringDoc OpenAPI. After adding the dependency, verify that Swagger UI is accessible and accurately reflects your endpoints.

8. **Test your endpoints** using `curl` or Postman. Include example requests and responses in your submission.
