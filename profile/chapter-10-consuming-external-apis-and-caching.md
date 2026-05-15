# Chapter 10: Consuming External APIs and Caching

---

## 10.1 Reaching Beyond Your Application

Every application we have built so far has been self-contained — our controllers talk to our services, our services talk to our repositories, and our repositories talk to our database. But modern web applications rarely exist in isolation. They fetch weather forecasts from weather services, convert currencies using exchange rate APIs, verify addresses through geolocation providers, process payments through payment gateways, and integrate with dozens of other external services.

This chapter covers two closely related topics: how to call external REST APIs from your Spring Boot application, and how to cache the results to avoid redundant calls. We will learn how to use `RestClient` — Spring Boot's modern, synchronous HTTP client — to make GET, POST, PUT, and DELETE requests, how to handle errors and timeouts, how to map JSON responses to Java objects, and how to use Spring Boot's declarative `@HttpExchange` interfaces for cleaner API client definitions. On the caching side, we will learn how Spring's cache abstraction works, how to apply `@Cacheable`, `@CacheEvict`, and `@CachePut`, and how to configure real cache providers like Caffeine and Redis.

---

## 10.2 HTTP Clients in Spring Boot

Spring Boot has provided several HTTP clients over the years, each representing a different generation of API design:

| Client | Status | Notes |
|--------|--------|-------|
| `RestTemplate` | Legacy (opt-in auto-configuration) | Synchronous, verbose. Auto-configuration is no longer enabled by default in Spring Boot 4. |
| `WebClient` | Active | Reactive/non-blocking. Requires the Spring WebFlux dependency. Best for reactive applications. |
| `RestClient` | **Recommended** | Synchronous, fluent API. Modern replacement for `RestTemplate`. Introduced in Spring Boot 3.2. |
| `@HttpExchange` interfaces | **Recommended (declarative)** | Interface-based, like Spring Data repositories for HTTP. Spring generates the implementation. |

For new projects, use `RestClient` for imperative HTTP calls or `@HttpExchange` interfaces for a declarative approach. This chapter covers both.

---

## 10.3 Making HTTP Requests with RestClient

### Creating a RestClient Bean

`RestClient` is created through a builder with a base URL, default headers, and timeout configuration:

```java
@Configuration
public class RestClientConfig {

    @Bean
    public RestClient weatherRestClient() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofSeconds(5));
        factory.setReadTimeout(Duration.ofSeconds(10));

        return RestClient.builder()
                .baseUrl("https://api.weatherapi.com/v1")
                .defaultHeader("Accept", "application/json")
                .requestFactory(factory)
                .build();
    }
}
```

The `SimpleClientHttpRequestFactory` configures connection and read timeouts. A 5-second connection timeout means the client will give up if it cannot establish a connection within 5 seconds. A 10-second read timeout means it will give up if the server does not respond within 10 seconds after the connection is established. Always set timeouts — without them, a slow or unresponsive external service can block your threads indefinitely.

### GET Requests

```java
@Service
public class WeatherService {

    private final RestClient restClient;

    public WeatherService(RestClient restClient) {
        this.restClient = restClient;
    }

    public WeatherResponse getWeather(String city) {
        return restClient.get()
                .uri("/weather?city={city}", city)
                .retrieve()
                .body(WeatherResponse.class);
    }
}
```

The fluent API reads like a sentence: build a GET request, set the URI with a path variable, retrieve the response, and deserialize the body into a `WeatherResponse` object. Jackson handles the JSON-to-Java conversion automatically.

### POST Requests

```java
public OrderConfirmation placeOrder(OrderRequest order) {
    return restClient.post()
            .uri("/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .body(order)
            .retrieve()
            .body(OrderConfirmation.class);
}
```

The `.body(order)` call serializes the `OrderRequest` object to JSON and sends it as the request body. The `.contentType(MediaType.APPLICATION_JSON)` header tells the server that the body is JSON.

### PUT and DELETE Requests

```java
public void updateUser(Long id, UserDto user) {
    restClient.put()
            .uri("/users/{id}", id)
            .contentType(MediaType.APPLICATION_JSON)
            .body(user)
            .retrieve()
            .toBodilessEntity();
}

public void deleteUser(Long id) {
    restClient.delete()
            .uri("/users/{id}", id)
            .retrieve()
            .toBodilessEntity();
}
```

`.toBodilessEntity()` retrieves the response as a `ResponseEntity<Void>` — useful when you do not need the response body but still want access to the status code and headers.

### URI Variables and Query Parameters

```java
// Path variables — values are substituted into {placeholders}
restClient.get()
        .uri("/users/{id}/posts/{postId}", userId, postId)
        .retrieve()
        .body(Post.class);

// Query parameters — built with a URI builder
restClient.get()
        .uri(uriBuilder -> uriBuilder
                .path("/search")
                .queryParam("q", searchTerm)
                .queryParam("page", page)
                .queryParam("size", size)
                .build())
        .retrieve()
        .body(SearchResult.class);
```

For simple path variables, the inline `{placeholder}` syntax is sufficient. For complex query strings with optional parameters, the `UriBuilder` lambda gives you full control.

---

## 10.4 Handling Responses and Errors

External APIs can fail for many reasons — the network is unreachable, the server is overloaded, the API key is invalid, the rate limit is exceeded. Your application must handle these failures gracefully rather than crashing or returning a stack trace.

### Accessing the Full Response

When you need the status code or headers alongside the body, use `.toEntity()`:

```java
public WeatherResponse getWeatherSafe(String city) {
    ResponseEntity<WeatherResponse> response = restClient.get()
            .uri("/weather?city={city}", city)
            .retrieve()
            .toEntity(WeatherResponse.class);

    if (response.getStatusCode().is2xxSuccessful() && response.getBody() != null) {
        return response.getBody();
    }

    throw new ExternalApiException("Failed to fetch weather data");
}
```

### Declarative Error Handling with Status Handlers

`RestClient` supports `.onStatus()` handlers that let you react to specific HTTP status codes declaratively:

```java
public WeatherResponse getWeather(String city) {
    return restClient.get()
            .uri("/weather?city={city}", city)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                throw new CityNotFoundException("City not found: " + city);
            })
            .onStatus(HttpStatusCode::is5xxServerError, (request, response) -> {
                throw new ExternalApiException(
                        "Weather API server error: " + response.getStatusCode());
            })
            .body(WeatherResponse.class);
}
```

The `.onStatus()` method registers a handler for a category of HTTP status codes. If the response matches, the handler executes instead of returning the body. This keeps error handling close to the request without cluttering the calling code with try-catch blocks.

### Handling Network Errors

Status handlers cover HTTP-level errors (4xx, 5xx), but network-level problems — connection refused, DNS resolution failure, timeouts — throw exceptions that require try-catch:

```java
public WeatherResponse getWeatherWithFallback(String city) {
    try {
        return restClient.get()
                .uri("/weather?city={city}", city)
                .retrieve()
                .body(WeatherResponse.class);
    } catch (ResourceAccessException e) {
        log.error("Cannot reach weather API: {}", e.getMessage());
        return WeatherResponse.fallback(city);
    }
}
```

`ResourceAccessException` is the exception Spring throws for I/O errors — connection timeouts, connection refused, and similar network-level failures. The fallback pattern shown here returns a default response instead of propagating the error, which is appropriate when the external data is supplementary (weather on a dashboard) rather than critical (payment processing).

---

## 10.5 Mapping JSON to Java Objects

Spring Boot uses Jackson to convert between JSON and Java objects automatically. When `RestClient` receives a JSON response, Jackson deserializes it into the class you specify. You just need a Java class whose field names match the JSON field names.

### Using Java Records

Java records are ideal for response DTOs — they are concise, immutable, and require no boilerplate:

```java
public record WeatherResponse(
    String city,
    double temperature,
    String description,
    Wind wind
) {
    public record Wind(double speed, String direction) {}
}
```

### Mapping Nested JSON

Given a JSON response with nested objects:

```json
{
    "location": {
        "name": "Tbilisi",
        "country": "Georgia"
    },
    "current": {
        "temp_c": 25.0,
        "condition": {
            "text": "Sunny"
        }
    }
}
```

Map it to nested records, using `@JsonProperty` when the JSON field name does not follow Java naming conventions:

```java
public record WeatherApiResponse(
    Location location,
    Current current
) {
    public record Location(String name, String country) {}
    public record Current(
        @JsonProperty("temp_c") double tempCelsius,
        Condition condition
    ) {}
    public record Condition(String text) {}
}
```

### Ignoring Unknown Fields

External APIs often return many more fields than you need. By default, Jackson throws an exception if the JSON contains fields that do not exist in your Java class. You can disable this globally:

```properties
spring.jackson.deserialization.fail-on-unknown-properties=false
```

Or per class:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record WeatherResponse(String city, double temperature) {}
```

### Deserializing Collections

When the API returns a JSON array, use `ParameterizedTypeReference` to preserve the generic type information:

```java
List<UserDto> users = restClient.get()
        .uri("/users")
        .retrieve()
        .body(new ParameterizedTypeReference<List<UserDto>>() {});
```

This is necessary because Java's type erasure removes generic type information at runtime. `ParameterizedTypeReference` captures the full `List<UserDto>` type so Jackson knows what to deserialize each element into.

---

## 10.6 Declarative HTTP Clients with @HttpExchange

`RestClient` gives you full control over HTTP requests, but for straightforward API integrations, the imperative style involves repetitive code — build the URI, call retrieve, deserialize the body. Spring Framework 7 introduces a declarative alternative: define your API client as a Java interface, annotate its methods, and Spring generates the implementation at runtime. Think of it as Spring Data repositories for HTTP.

### Defining an HTTP Interface

```java
@HttpExchange("/weather")
public interface WeatherClient {

    @GetExchange
    WeatherResponse getWeather(@RequestParam String city);

    @GetExchange("/{id}")
    WeatherResponse getWeatherById(@PathVariable Long id);
}
```

The interface declares what endpoints exist and what types they work with. No implementation class, no request building, no response parsing.

### Registering the Client with @ImportHttpServices

In Spring Boot 4, the `@ImportHttpServices` annotation registers your interface as a Spring bean and assigns it to a named group:

```java
@Configuration
@ImportHttpServices(group = "weather", types = WeatherClient.class)
public class HttpClientConfig {

    @Bean
    public RestClientHttpServiceGroupConfigurer weatherConfigurer() {
        return groups -> {
            groups.filterByName("weather")
                    .forEachClient((group, clientBuilder) -> {
                        clientBuilder.baseUrl("https://api.weatherapi.com/v1");
                        clientBuilder.defaultHeader("Accept", "application/json");
                    });
        };
    }
}
```

`@ImportHttpServices` tells Spring to scan the `WeatherClient` interface, generate a proxy implementation backed by `RestClient`, and register it as a bean in the `weather` group. The `RestClientHttpServiceGroupConfigurer` sets the base URL and default headers for all clients in that group. If you add more interfaces to the same group later, they automatically inherit the same configuration.

### Using the Client

Inject and use the client like any other Spring bean:

```java
@Service
public class WeatherService {

    private final WeatherClient weatherClient;

    public WeatherService(WeatherClient weatherClient) {
        this.weatherClient = weatherClient;
    }

    public WeatherResponse getWeather(String city) {
        return weatherClient.getWeather(city);
    }
}
```

### When to Use Which Approach

Use `RestClient` directly when you need fine-grained control over the request — custom headers per request, status handlers, response entity access, or dynamic URIs. Use `@HttpExchange` interfaces when the API integration is straightforward and you want the cleanest possible code. Both approaches are fully supported and can coexist in the same application.

---

## 10.7 Introduction to Caching

Calling an external API on every request is wasteful when the data does not change frequently. Weather data is valid for 10 minutes. Exchange rates update hourly. Product catalogs change daily. Fetching the same data repeatedly wastes network bandwidth, increases latency, and risks hitting API rate limits.

**Caching** stores the result of an expensive operation so that subsequent requests for the same data are served from the cache — a local, in-memory store — instead of calling the external service again.

### Enabling Caching

Add `@EnableCaching` to a configuration class:

```java
@Configuration
@EnableCaching
public class CacheConfig {
    // Spring Boot auto-configures a simple ConcurrentHashMap-based cache
}
```

This single annotation activates Spring's cache abstraction. By default, Spring Boot uses a `ConcurrentHashMap` as the cache store — no additional dependencies needed. This is sufficient for development, but for production you will want a proper cache provider (covered in section 10.10).

### @Cacheable — Cache Method Results

The `@Cacheable` annotation tells Spring to cache the return value of a method. On the first call, the method executes and the result is stored. On subsequent calls with the same arguments, the cached result is returned without executing the method body:

```java
@Service
public class WeatherService {

    private final RestClient restClient;

    public WeatherService(RestClient restClient) {
        this.restClient = restClient;
    }

    @Cacheable(value = "weather", key = "#city")
    public WeatherResponse getWeather(String city) {
        log.info("Calling external weather API for city: {}", city);
        return restClient.get()
                .uri("/weather?city={city}", city)
                .retrieve()
                .body(WeatherResponse.class);
    }
}
```

`value = "weather"` names the cache. You can have multiple caches for different types of data. `key = "#city"` defines the cache key using a SpEL expression — each city gets its own cache entry. If `getWeather("Tbilisi")` is called and the cache already has an entry for `"Tbilisi"`, the method body is skipped entirely and the cached `WeatherResponse` is returned. You can verify this by watching the log — the "Calling external weather API" message will appear on the first call but not on subsequent ones.

---

## 10.8 Cache Eviction and Updates

Cached data eventually becomes stale. You need ways to remove or refresh it.

### @CacheEvict — Remove Entries

`@CacheEvict` removes entries from the cache. Use it when the underlying data has changed or when you want to force a refresh:

```java
@CacheEvict(value = "weather", key = "#city")
public void refreshWeather(String city) {
    log.info("Cache evicted for city: {}", city);
}

@CacheEvict(value = "weather", allEntries = true)
public void refreshAllWeather() {
    log.info("All weather cache entries evicted");
}
```

`allEntries = true` clears the entire cache, not just a single key. This is useful for scheduled cleanup — for example, evicting all weather data every 10 minutes:

```java
@CacheEvict(value = "weather", allEntries = true)
@Scheduled(fixedRate = 600000)
public void evictAllWeatherCache() {
    log.info("Scheduled cache eviction");
}
```

The `@Scheduled` annotation (which requires `@EnableScheduling` on a configuration class) runs the method at a fixed interval. Combined with `@CacheEvict`, it provides automatic cache expiration.

### @CachePut — Execute and Update

`@CachePut` always executes the method and stores the result in the cache, replacing any existing entry. Unlike `@Cacheable`, it never skips the method:

```java
@CachePut(value = "weather", key = "#city")
public WeatherResponse forceRefreshWeather(String city) {
    return restClient.get()
            .uri("/weather?city={city}", city)
            .retrieve()
            .body(WeatherResponse.class);
}
```

This is useful for "refresh" buttons — the user explicitly wants fresh data, so you call the API and update the cache simultaneously.

### Conditional Caching

You can control when caching applies using `condition` and `unless`:

```java
@Cacheable(value = "weather", key = "#city", condition = "#city.length() > 2")
public WeatherResponse getWeather(String city) {
    return fetchFromApi(city);
}

@Cacheable(value = "weather", key = "#city", unless = "#result == null")
public WeatherResponse getWeatherOrNull(String city) {
    return fetchFromApi(city);
}
```

`condition` is evaluated before the method executes — if it is `false`, the method runs without caching. `unless` is evaluated after the method executes — if it is `true`, the result is not cached. The second example prevents caching null results, which is important when a null response indicates a temporary failure that should be retried.

---

## 10.9 When to Cache (and When Not To)

Caching improves performance dramatically when used appropriately, but it introduces complexity — stale data, memory consumption, and cache invalidation logic. Use it deliberately.

### Good Candidates

| Scenario | Why |
|----------|-----|
| External API responses | Reduce network calls, avoid rate limits. |
| Database queries that rarely change | Skip expensive queries for stable data. |
| Computed or aggregated data | Save CPU time on recalculations. |
| Configuration or reference data | Small dataset, read frequently, changes rarely. |

### Poor Candidates

| Scenario | Why |
|----------|-----|
| Frequently changing data | Cache becomes stale quickly, causing inconsistencies. |
| User-specific, security-sensitive data | Risk of serving one user's data to another. |
| Large datasets | Consumes too much memory. |
| Write-heavy operations | Cache invalidation becomes a constant burden. |

### Best Practices

**Set appropriate TTL (Time-To-Live).** Never cache data forever. External API data becomes stale, and unbounded caches grow until they consume all available memory.

**Use meaningful cache names.** `"weather"` and `"exchangeRates"` are clear. `"cache1"` and `"data"` are not.

**Do not cache null results** unless you intend to. Use `unless = "#result == null"` to prevent caching failures.

**Monitor cache hit/miss ratios.** A cache that never gets hit is wasting memory. A cache that is always stale is adding complexity for no benefit.

**Handle cache failures gracefully.** If the cache provider is down, the application should still work — just slower.

---

## 10.10 Cache Providers

Spring Boot's default cache implementation uses a `ConcurrentHashMap`, which has no TTL support, no size limits, and no eviction policies. For production, you need a real cache provider.

### Caffeine (Recommended for Single-Instance Applications)

Caffeine is a high-performance, in-memory caching library. It supports time-based eviction, size-based eviction, and statistics — everything the default `ConcurrentHashMap` lacks.

Add the dependency:

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

Configure in `application.properties`:

```properties
spring.cache.type=caffeine
spring.cache.caffeine.spec=maximumSize=500,expireAfterWrite=10m
```

Or configure programmatically for more control:

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("weather", "exchangeRates");
        cacheManager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(500)
                .expireAfterWrite(Duration.ofMinutes(10))
                .recordStats());
        return cacheManager;
    }
}
```

`maximumSize(500)` evicts the least-recently-used entries when the cache exceeds 500 entries. `expireAfterWrite(Duration.ofMinutes(10))` automatically expires entries 10 minutes after they were written. `recordStats()` enables hit/miss statistics that you can monitor through Spring Boot Actuator. Caffeine is the right choice for most single-instance applications — it is fast, lightweight, and runs in-process with no external infrastructure.

### Redis (Recommended for Distributed Applications)

When your application runs on multiple instances behind a load balancer, an in-process cache like Caffeine does not help — each instance has its own cache, and a request that hits instance A will not benefit from data cached on instance B. Redis solves this by providing a shared, external cache that all instances can read from and write to.

Add the dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

Configure in `application.properties`:

```properties
spring.cache.type=redis
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.cache.redis.time-to-live=600000
```

Or configure programmatically:

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .serializeValuesWith(
                        RedisSerializationContext.SerializationPair
                                .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(config)
                .build();
    }
}
```

Redis requires a running Redis server — it is external infrastructure that you must provision and manage. Use it when you need distributed caching across multiple application instances or when your cache must survive application restarts.

### Comparison

| Feature | ConcurrentHashMap (default) | Caffeine | Redis |
|---------|---------------------------|----------|-------|
| **Setup** | Zero configuration | Add dependency | Requires Redis server |
| **TTL support** | No | Yes | Yes |
| **Size limits** | No | Yes | Yes (configurable) |
| **Distributed** | No | No | Yes |
| **Performance** | Fast (in-process) | Very fast (in-process) | Fast (network hop) |
| **Best for** | Development and testing | Single-instance production | Multi-instance production |

---

## 10.11 Putting It All Together: A Weather Dashboard

Here is a complete example that combines `RestClient`, error handling, caching, and a FreeMarker template:

### RestClient Configuration

```java
@Configuration
public class RestClientConfig {

    @Bean
    public RestClient weatherRestClient() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofSeconds(5));
        factory.setReadTimeout(Duration.ofSeconds(10));

        return RestClient.builder()
                .baseUrl("https://api.weatherapi.com/v1")
                .defaultHeader("Accept", "application/json")
                .requestFactory(factory)
                .build();
    }
}
```

### Cache Configuration

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("weather");
        cacheManager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(200)
                .expireAfterWrite(Duration.ofMinutes(15))
                .recordStats());
        return cacheManager;
    }
}
```

### Weather Service

```java
@Service
public class WeatherService {

    private static final Logger log = LoggerFactory.getLogger(WeatherService.class);
    private final RestClient restClient;

    public WeatherService(RestClient restClient) {
        this.restClient = restClient;
    }

    @Cacheable(value = "weather", key = "#city")
    public WeatherResponse getWeather(String city) {
        log.info("Fetching weather for {} from external API", city);
        return restClient.get()
                .uri("/weather?city={city}", city)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
                    throw new CityNotFoundException("City not found: " + city);
                })
                .onStatus(HttpStatusCode::is5xxServerError, (req, res) -> {
                    throw new ExternalApiException("Weather API is unavailable");
                })
                .body(WeatherResponse.class);
    }

    @CacheEvict(value = "weather", key = "#city")
    public void evictWeatherCache(String city) {
        log.info("Evicted weather cache for {}", city);
    }
}
```

### Weather Controller

```java
@Controller
public class WeatherController {

    private final WeatherService weatherService;

    public WeatherController(WeatherService weatherService) {
        this.weatherService = weatherService;
    }

    @GetMapping("/weather")
    public String showWeather(@RequestParam(required = false) String city, Model model) {
        if (city != null && !city.isBlank()) {
            try {
                WeatherResponse weather = weatherService.getWeather(city);
                model.addAttribute("weather", weather);
            } catch (CityNotFoundException e) {
                model.addAttribute("error", "City not found: " + city);
            } catch (ExternalApiException e) {
                model.addAttribute("error", "Weather service is temporarily unavailable.");
            }
        }
        model.addAttribute("city", city);
        return "weather";
    }
}
```

### The FreeMarker Template

```html
<#import "/spring.ftl" as spring>
<#import "layout.ftlh" as layout>

<@layout.page title="Weather Dashboard">
    <h1>Weather Dashboard</h1>

    <form action="<@spring.url '/weather'/>" method="get">
        <input type="text" name="city" value="${(city)!''}" placeholder="Enter city name"/>
        <button type="submit">Search</button>
    </form>

    <#if error??>
        <p class="error">${error}</p>
    </#if>

    <#if weather??>
        <div class="weather-card">
            <h2>${weather.city}</h2>
            <p>Temperature: ${weather.temperature}°C</p>
            <p>Condition: ${weather.description}</p>
            <p>Wind: ${weather.wind.speed} km/h, ${weather.wind.direction}</p>
        </div>
    </#if>
</@layout.page>
```

---

## Summary

This chapter covered consuming external REST APIs and caching their responses in a Spring Boot application.

**`RestClient`** is Spring Boot's modern, synchronous HTTP client. Its fluent builder API supports GET, POST, PUT, and DELETE requests with path variables, query parameters, request bodies, and custom headers. Timeouts should always be configured through a `SimpleClientHttpRequestFactory`.

**Error handling** operates at two levels. HTTP-level errors (4xx, 5xx) are handled declaratively with `.onStatus()` handlers. Network-level errors (timeouts, connection refused) require try-catch blocks around `ResourceAccessException`.

**JSON mapping** uses Jackson to convert between JSON responses and Java objects. Java records provide concise, immutable DTOs for response mapping. `@JsonProperty` handles name mismatches, `@JsonIgnoreProperties(ignoreUnknown = true)` tolerates extra fields, and `ParameterizedTypeReference` handles generic collections.

**`@HttpExchange` interfaces** provide a declarative alternative to `RestClient`. You define an interface with annotated methods, register it with `@ImportHttpServices`, and Spring generates the implementation. This approach minimizes boilerplate for straightforward API integrations.

**Spring's cache abstraction** (`@EnableCaching`) provides annotation-driven caching. `@Cacheable` caches method results and skips execution on cache hits. `@CacheEvict` removes entries when data changes. `@CachePut` always executes and updates the cache. `condition` and `unless` control when caching applies.

**Cache providers** range from the default `ConcurrentHashMap` (development only) to **Caffeine** (high-performance, in-process, ideal for single-instance applications) to **Redis** (distributed, shared across instances, ideal for multi-instance production deployments).

---

## Resources

- [RestClient — Spring Framework Reference](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-restclient)
- [HTTP Interface Clients — Spring Framework Reference](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-interface)
- [Caching — Spring Boot Reference](https://docs.spring.io/spring-boot/reference/io/caching.html)
- [Spring Boot Cache with Caffeine — Baeldung](https://www.baeldung.com/spring-boot-caffeine-cache)
- [Caffeine Cache](https://github.com/ben-manes/caffeine)
- [Spring Data Redis — Spring Boot Reference](https://docs.spring.io/spring-boot/reference/data/nosql.html#data.nosql.redis)
- [WeatherAPI](https://www.weatherapi.com/)
- [OpenWeatherMap API](https://openweathermap.org/api)

---

## Lab Assignment: Build a Weather Dashboard with Cached API Responses

Build a Spring Boot web application that consumes a public weather API and displays weather information, with caching to minimize API calls.

**Requirements:**

1. **Consume a public weather API** (such as [WeatherAPI](https://www.weatherapi.com/) or [OpenWeatherMap](https://openweathermap.org/api)) using `RestClient`. Sign up for a free API key. Fetch current weather data for a given city.

2. **Create a Weather Dashboard page** (using FreeMarker) with a search form where the user enters a city name. Display the current temperature, weather condition, humidity, and wind speed. Show a user-friendly error message if the city is not found or the API is unreachable.

3. **Implement caching** so that weather data for a city is cached for at least 10 minutes using `@Cacheable`. Add a "Refresh" button that evicts the cache for a specific city using `@CacheEvict`. Use Caffeine as the cache provider with a maximum of 100 entries.

4. **Handle errors gracefully:** configure a 5-second connection timeout and 10-second read timeout, handle invalid city names (4xx errors) and API server errors (5xx errors) with `.onStatus()` handlers, and catch `ResourceAccessException` for network failures.

5. *(Bonus)* Add a "Recent Searches" section that shows the last 5 cities searched, stored in the HTTP session.

6. *(Bonus)* Define a second API integration using the `@HttpExchange` declarative interface approach (e.g., exchange rates or news headlines) to practice both styles of HTTP client.
