# Chapter 7: Testing Web Applications

---

## 7.1 Why Testing Matters

In the previous chapters, we built controllers, templates, entities, repositories, REST APIs, and configuration profiles. We verified that things worked by running the application and clicking through the browser, or by sending requests with curl. This approach — manual testing — works when the application is small. It stops working the moment the application grows beyond a few features.

Manual testing is slow, unreliable, and does not scale. You cannot manually re-check every feature every time you change a line of code. Automated tests can. They run in seconds, they catch regressions immediately, and they give you the confidence to refactor without fear of breaking something.

This chapter introduces the testing tools that Spring Boot provides. We will learn the testing pyramid and the different levels of testing, how to write unit tests with JUnit 5 and Mockito, how to test Spring components in isolation using slice annotations, how to test REST APIs and form submissions with MockMvc, and how to configure test-specific profiles and databases. By the end, you will have a practical testing strategy for your application.

Spring Boot includes all the necessary testing dependencies — JUnit 5, Mockito, AssertJ, MockMvc, and more — through the `spring-boot-starter-test` dependency.

---

## 7.2 The Testing Pyramid

Not all tests serve the same purpose. The testing pyramid describes three levels, ordered by speed, scope, and quantity:

**Unit tests** sit at the base of the pyramid. They test a single class or method in complete isolation from the rest of the application. Dependencies are replaced with mocks. Unit tests are fast (milliseconds each), numerous (you should have hundreds of them), and cheap to write and maintain.

**Integration tests** sit in the middle. They test how multiple components work together — a service talking to a real repository, a controller connected to the Spring context. They are slower than unit tests because they may start a Spring context or interact with a database, but they catch problems that unit tests miss — wiring errors, configuration issues, query bugs.

**End-to-end tests** sit at the top. They test the entire application from the user's perspective — launching a browser, filling in forms, clicking buttons. They are the slowest, the most brittle, and the most expensive to maintain. They provide the highest confidence but should be used sparingly.

The pyramid shape is intentional: write many unit tests, a moderate number of integration tests, and a small number of end-to-end tests. This chapter focuses on the first two levels.

---

## 7.3 Unit Testing with JUnit 5

JUnit 5 is the standard testing framework for Java. Spring Boot includes it by default through `spring-boot-starter-test`, so no additional dependencies are needed.

### Basic Test Structure

A test class is a plain Java class. Each test method is annotated with `@Test`:

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    @Test
    void shouldAddTwoNumbers() {
        Calculator calculator = new Calculator();
        int result = calculator.add(2, 3);
        assertEquals(5, result);
    }

    @Test
    void shouldThrowExceptionWhenDividingByZero() {
        Calculator calculator = new Calculator();
        assertThrows(ArithmeticException.class, () -> calculator.divide(10, 0));
    }
}
```

Each test method follows a three-part structure, sometimes called **Arrange-Act-Assert**: set up the objects you need, execute the operation you are testing, and verify the result. The test method name should describe what behavior is being verified — `shouldAddTwoNumbers` is far more informative than `testAdd`.

### Common Assertions

```java
assertEquals(expected, actual);              // Values are equal
assertNotEquals(unexpected, actual);         // Values differ
assertTrue(condition);                       // Condition is true
assertFalse(condition);                      // Condition is false
assertNull(value);                           // Value is null
assertNotNull(value);                        // Value is not null
assertThrows(ExceptionType.class, () -> {}); // Exception is thrown
assertAll(                                   // Multiple assertions grouped
    () -> assertEquals("John", user.getName()),
    () -> assertEquals(25, user.getAge())
);
```

`assertAll` is particularly useful — it runs all assertions even if one fails, so you see all failures at once rather than fixing them one at a time.

### Lifecycle Annotations

JUnit 5 provides annotations for setup and teardown code that runs at specific points in the test lifecycle:

| Annotation | When It Runs |
|------------|-------------|
| `@BeforeEach` | Before each test method. Use for creating fresh objects. |
| `@AfterEach` | After each test method. Use for cleanup. |
| `@BeforeAll` | Once before all tests in the class (method must be `static`). |
| `@AfterAll` | Once after all tests in the class (method must be `static`). |
| `@DisplayName` | Provides a human-readable name for the test. |
| `@Disabled` | Skips a test. |
| `@ParameterizedTest` | Runs the same test with different inputs. |

```java
class ProductServiceTest {

    private ProductService productService;

    @BeforeEach
    void setUp() {
        productService = new ProductService(new FakeProductRepository());
    }

    @Test
    void shouldFindProductById() {
        Product product = productService.findById(1L);
        assertNotNull(product);
        assertEquals("Laptop", product.getName());
    }
}
```

`@BeforeEach` ensures that each test starts with a fresh instance of `ProductService`. This isolation is important — tests should never depend on the state left behind by previous tests.

---

## 7.4 Mocking Dependencies with Mockito

A unit test tests a single class in isolation. But most classes depend on other classes — a `ProductService` depends on a `ProductRepository`, an `OrderService` depends on both an `OrderRepository` and a `ProductRepository`. If you use the real dependencies, you are no longer testing in isolation — a bug in the repository could cause a service test to fail, and you would be testing against a real database.

**Mockito** solves this by letting you create fake objects — mocks — that stand in for the real dependencies. You tell the mock what to return when a specific method is called, and you verify that the class under test interacted with the mock correctly.

### Creating Mocks with Annotations

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class ProductServiceTest {

    @Mock
    private ProductRepository productRepository;

    @InjectMocks
    private ProductService productService;

    @Test
    void shouldReturnProductWhenFound() {
        // Arrange
        Product mockProduct = new Product(1L, "Laptop", 999.99);
        when(productRepository.findById(1L)).thenReturn(Optional.of(mockProduct));

        // Act
        Product result = productService.findById(1L);

        // Assert
        assertNotNull(result);
        assertEquals("Laptop", result.getName());
        assertEquals(999.99, result.getPrice());

        // Verify
        verify(productRepository, times(1)).findById(1L);
    }

    @Test
    void shouldThrowExceptionWhenProductNotFound() {
        when(productRepository.findById(99L)).thenReturn(Optional.empty());

        assertThrows(RuntimeException.class, () -> productService.findById(99L));
    }
}
```

Let us break down the mechanics.

`@ExtendWith(MockitoExtension.class)` activates Mockito's JUnit 5 integration. Without this, the `@Mock` and `@InjectMocks` annotations would have no effect.

`@Mock` creates a mock implementation of `ProductRepository`. The mock does nothing by default — all methods return null, empty collections, or zero unless you tell them otherwise.

`@InjectMocks` creates a real instance of `ProductService` and injects the mocks into it through constructor injection. This is why constructor injection (which we have used throughout this course) matters — it makes the class trivially testable.

`when(productRepository.findById(1L)).thenReturn(Optional.of(mockProduct))` programs the mock: "when `findById` is called with `1L`, return this product." This is the Arrange step.

`verify(productRepository, times(1)).findById(1L)` checks that the service actually called the repository's `findById` method exactly once. This is useful for verifying side effects — for example, ensuring that a save operation was performed.

### Key Mockito Methods

| Method | Purpose |
|--------|---------|
| `when(...).thenReturn(...)` | Define what a mock returns when called. |
| `when(...).thenThrow(...)` | Define what exception a mock throws. |
| `verify(mock, times(n)).method()` | Verify a method was called exactly n times. |
| `verify(mock, never()).method()` | Verify a method was never called. |
| `any()`, `anyLong()`, `anyString()` | Argument matchers for flexible matching. |

---

## 7.5 Writing Testable Code

Testing is easier when the code is designed for it. A few principles make the difference:

**Use constructor injection.** We have used it throughout this course, and this is one of the main reasons — constructor injection makes it trivial to substitute mocks for real dependencies. If a class creates its own dependencies with `new`, you cannot replace them in a test.

```java
// Testable: dependencies injected via constructor
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final NotificationService notificationService;

    public OrderService(OrderRepository orderRepository,
                        NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.notificationService = notificationService;
    }
}

// Not testable: creates dependencies internally
@Service
public class OrderService {

    public void createOrder(Order order) {
        OrderRepository repo = new OrderRepository(); // cannot mock this
        repo.save(order);
    }
}
```

**Depend on interfaces, not concrete classes.** If `OrderService` depends on `OrderRepository` (an interface), you can mock it. If it depends on `OrderRepositoryImpl` (a concrete class), mocking is harder and your code is less flexible.

**Keep methods small and focused.** A method that does one thing is easy to test. A method that does ten things requires a complex test with many assertions and mocks.

**Avoid static methods in business logic.** Static methods cannot be mocked with standard Mockito. If your business logic calls `LocalDateTime.now()`, you cannot control the time in your tests. Inject a `Clock` instead.

---

## 7.6 Integration Testing with @SpringBootTest

Unit tests verify that individual classes work correctly in isolation. Integration tests verify that the pieces work together — that the service is correctly wired to the repository, that the repository generates valid SQL, that the configuration is correct.

`@SpringBootTest` starts the full Spring application context, creating all beans and wiring them together just as the real application would:

```java
@SpringBootTest
class ProductServiceIntegrationTest {

    @Autowired
    private ProductService productService;

    @Autowired
    private ProductRepository productRepository;

    @BeforeEach
    void setUp() {
        productRepository.deleteAll();
    }

    @Test
    void shouldSaveAndRetrieveProduct() {
        Product product = new Product();
        product.setName("Test Product");
        product.setPrice(29.99);

        Product saved = productService.save(product);

        assertNotNull(saved.getId());
        assertEquals("Test Product", saved.getName());

        Product found = productService.findById(saved.getId());
        assertEquals("Test Product", found.getName());
    }
}
```

This test exercises the full chain: the service calls the repository, the repository talks to the database (H2 by default in tests), and the data is persisted and retrieved. If the entity mapping is wrong, the query is malformed, or the transaction management is broken, this test will catch it.

The tradeoff is speed. `@SpringBootTest` loads the entire application context, which takes seconds. For a few tests this is fine, but if you have hundreds of integration tests that each start a fresh context, your test suite will be painfully slow. This is where slice tests help.

---

## 7.7 Slice Tests

Spring Boot provides slice annotations that load only a portion of the application context — just enough to test a specific layer. They are faster than `@SpringBootTest` because they skip everything that is not relevant.

### @WebMvcTest — Testing the Web Layer

`@WebMvcTest` loads only the web layer — controllers, filters, and exception handlers. It does not start a real server and does not create service or repository beans. Dependencies must be provided as mocks.

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private ProductService productService;

    @Test
    void shouldReturnProductListPage() throws Exception {
        List<Product> products = List.of(
                new Product(1L, "Laptop", 999.99),
                new Product(2L, "Phone", 699.99)
        );
        when(productService.findAll()).thenReturn(products);

        mockMvc.perform(get("/products"))
                .andExpect(status().isOk())
                .andExpect(view().name("products/list"))
                .andExpect(model().attributeExists("products"))
                .andExpect(model().attribute("products", hasSize(2)));
    }
}
```

Notice `@MockitoBean` instead of the older `@MockBean`. In Spring Boot 4, `@MockBean` has been removed. Its replacement, `@MockitoBean`, comes from Spring Framework itself (the `org.springframework.test.context.bean.override.mockito` package) and provides the same functionality — it creates a Mockito mock and registers it in the Spring application context, replacing any real bean of the same type.

`@WebMvcTest(ProductController.class)` tells Spring to load only the `ProductController` and the web infrastructure it needs. The `ProductService` is not loaded — we provide a mock through `@MockitoBean`. This means the test is fast (no database, no service layer) and focused (it tests only the controller's behavior).

### @DataJpaTest — Testing the Data Layer

`@DataJpaTest` loads only JPA-related components — entities, repositories, and an embedded database. It is ideal for testing that your entity mappings are correct, your derived query methods work, and your custom `@Query` methods produce valid SQL.

```java
@DataJpaTest
class ProductRepositoryTest {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldFindProductsByPriceGreaterThan() {
        Product cheap = new Product();
        cheap.setName("Pen");
        cheap.setPrice(1.99);
        entityManager.persist(cheap);

        Product expensive = new Product();
        expensive.setName("Laptop");
        expensive.setPrice(999.99);
        entityManager.persist(expensive);

        entityManager.flush();

        List<Product> results = productRepository.findByPriceGreaterThan(100.0);

        assertEquals(1, results.size());
        assertEquals("Laptop", results.get(0).getName());
    }
}
```

`@DataJpaTest` uses an embedded H2 database by default and rolls back each test's transaction automatically. This means each test starts with a clean database — no state leaks between tests. The `TestEntityManager` is a testing utility that provides a simpler API for persisting test data.

---

## 7.8 Testing REST APIs with MockMvc

`MockMvc` simulates HTTP requests without starting a real HTTP server. It lets you test controllers by sending requests and asserting on the response status, headers, body, and JSON structure.

### Testing GET Endpoints

```java
@WebMvcTest(ProductRestController.class)
class ProductRestControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private ProductService productService;

    @Test
    void shouldReturnAllProducts() throws Exception {
        List<Product> products = List.of(
                new Product(1L, "Laptop", 999.99),
                new Product(2L, "Phone", 699.99)
        );
        when(productService.findAll()).thenReturn(products);

        mockMvc.perform(get("/api/products")
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(2)))
                .andExpect(jsonPath("$[0].name").value("Laptop"))
                .andExpect(jsonPath("$[1].name").value("Phone"));
    }

    @Test
    void shouldReturn404WhenProductNotFound() throws Exception {
        when(productService.findById(99L))
                .thenThrow(new ProductNotFoundException("Product not found"));

        mockMvc.perform(get("/api/products/99"))
                .andExpect(status().isNotFound());
    }
}
```

The `jsonPath` method uses JsonPath expressions to assert on specific fields within the JSON response. `$` refers to the root of the JSON. `$[0].name` refers to the `name` field of the first element in a JSON array. `hasSize(2)` asserts that the array has exactly two elements.

### Testing POST Endpoints

```java
@Test
void shouldCreateProduct() throws Exception {
    Product saved = new Product(1L, "New Product", 49.99);
    when(productService.save(any(Product.class))).thenReturn(saved);

    mockMvc.perform(post("/api/products")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                        {
                            "name": "New Product",
                            "price": 49.99
                        }
                        """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("New Product"));
}
```

The `.content(...)` method provides the request body — in this case, a JSON string using a Java text block (the `"""..."""` syntax). The `any(Product.class)` matcher in the `when` clause matches any `Product` argument, which is appropriate here because the deserialized object will not be the exact same instance as `saved`.

### Testing Form Submissions

MockMvc is not limited to REST APIs — it can also test traditional form submissions from `@Controller` classes:

```java
@WebMvcTest(ProductController.class)
class ProductFormTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private ProductService productService;

    @Test
    void shouldCreateProductFromForm() throws Exception {
        mockMvc.perform(post("/products")
                        .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                        .param("name", "Test Product")
                        .param("price", "29.99")
                        .param("description", "A test product"))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/products"));

        verify(productService, times(1)).save(any(Product.class));
    }

    @Test
    void shouldRejectInvalidForm() throws Exception {
        mockMvc.perform(post("/products")
                        .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                        .param("name", "")
                        .param("price", "-5"))
                .andExpect(status().isOk())
                .andExpect(view().name("products/create"))
                .andExpect(model().hasErrors());

        verify(productService, never()).save(any(Product.class));
    }
}
```

The first test verifies that a valid form submission creates a product and redirects (the Post/Redirect/Get pattern from Chapter 3). The second test verifies that an invalid form (blank name, negative price) re-displays the form template with validation errors and does **not** call the service to save. The `verify(productService, never()).save(...)` assertion is crucial here — it confirms that the validation actually prevented the save.

---

## 7.9 Test Profiles and Embedded Databases

Integration tests that hit a real database need a predictable, isolated environment. You do not want tests running against your development database — they might interfere with your data, or your data might cause tests to fail unpredictably.

### Test Profile Configuration

Create a test-specific properties file in `src/test/resources/`:

```properties
# src/test/resources/application-test.properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
```

Activate the test profile in your test class with `@ActiveProfiles`:

```java
@SpringBootTest
@ActiveProfiles("test")
class ProductServiceIntegrationTest {
    // All tests run with the test profile configuration
}
```

This ensures that integration tests use an in-memory H2 database regardless of what the `dev` or `prod` profiles specify. The schema is created fresh on startup and dropped on shutdown, guaranteeing a clean state.

### @TestConfiguration

Sometimes you need beans that exist only in the test context — a fake email sender, a stub for an external service, or a differently configured component. `@TestConfiguration` defines beans that are added to the test context without affecting the main application:

```java
@TestConfiguration
class TestConfig {

    @Bean
    public NotificationService notificationService() {
        return new FakeNotificationService();
    }
}
```

```java
@SpringBootTest
@Import(TestConfig.class)
class OrderServiceIntegrationTest {
    // Uses FakeNotificationService instead of the real one
}
```

The `@Import` annotation explicitly adds the test configuration to the context. This is cleaner than using `@MockitoBean` when you want a deterministic fake implementation rather than a mock that requires programming with `when(...).thenReturn(...)`.

---

## 7.10 A Complete Test Suite Example

To see how the different test types work together, here is a complete set of tests for an order processing feature:

### Unit Test — Service Logic

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private ProductRepository productRepository;

    @InjectMocks
    private OrderService orderService;

    @Test
    @DisplayName("Should create order and update product stock")
    void shouldCreateOrderSuccessfully() {
        Product product = new Product(1L, "Laptop", 999.99);
        product.setStock(10);

        OrderItem item = new OrderItem(1L, 2);
        Order order = new Order();
        order.setItems(List.of(item));

        when(productRepository.findById(1L)).thenReturn(Optional.of(product));
        when(orderRepository.save(any(Order.class))).thenReturn(order);

        Order result = orderService.createOrder(order);

        assertNotNull(result);
        assertEquals(8, product.getStock());
        verify(productRepository).save(product);
        verify(orderRepository).save(order);
    }
}
```

This unit test verifies the business logic: when an order is placed for 2 units, the stock decreases from 10 to 8, both the product and the order are saved. No database is involved — everything is mocked.

### Slice Test — Repository

```java
@DataJpaTest
class ProductRepositoryTest {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldFindProductsByPriceGreaterThan() {
        Product cheap = new Product();
        cheap.setName("Pen");
        cheap.setPrice(1.99);
        entityManager.persist(cheap);

        Product expensive = new Product();
        expensive.setName("Laptop");
        expensive.setPrice(999.99);
        entityManager.persist(expensive);

        entityManager.flush();

        List<Product> results = productRepository.findByPriceGreaterThan(100.0);

        assertEquals(1, results.size());
        assertEquals("Laptop", results.get(0).getName());
    }
}
```

This test verifies that the derived query method `findByPriceGreaterThan` generates correct SQL and returns the right results. It uses a real (embedded) database but does not load any controllers or services.

### Slice Test — Controller

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private ProductService productService;

    @Test
    void shouldShowProductDetails() throws Exception {
        Product product = new Product(1L, "Laptop", 999.99);
        when(productService.findById(1L)).thenReturn(product);

        mockMvc.perform(get("/products/1"))
                .andExpect(status().isOk())
                .andExpect(view().name("products/detail"))
                .andExpect(model().attribute("product", product));
    }
}
```

This test verifies that the controller puts the correct product in the model and returns the right view name. The service is mocked — the test does not care how the product is fetched, only that the controller handles it correctly.

Together, these three tests cover the same feature at three different levels: the service's business logic, the repository's query correctness, and the controller's request handling. If any layer breaks, the appropriate test will catch it.

---

## Summary

This chapter covered the tools and techniques for testing Spring Boot applications at multiple levels.

**The testing pyramid** guides test strategy: many fast unit tests at the base, a moderate number of integration tests in the middle, and few slow end-to-end tests at the top.

**JUnit 5** provides the framework for writing and organizing tests. `@Test` marks test methods. `@BeforeEach` and `@AfterEach` handle setup and teardown. Assertions like `assertEquals`, `assertThrows`, and `assertAll` verify expected behavior.

**Mockito** creates mock objects that replace real dependencies in unit tests. `@Mock` creates mocks, `@InjectMocks` injects them into the class under test, `when(...).thenReturn(...)` programs mock behavior, and `verify` confirms that expected interactions occurred.

**Testable code** follows patterns that make testing easy: constructor injection, interface dependencies, small focused methods, and no internal object creation.

**`@SpringBootTest`** loads the full application context for integration testing. It is thorough but slow — use it when you need to verify that the full stack works together.

**Slice tests** load only part of the context. `@WebMvcTest` tests controllers with MockMvc, mocking service dependencies with `@MockitoBean`. `@DataJpaTest` tests repositories against an embedded database with automatic transaction rollback.

**MockMvc** simulates HTTP requests without starting a server, enabling tests for both REST APIs (asserting on JSON responses with `jsonPath`) and form submissions (asserting on redirects, view names, and model attributes).

**Test profiles** (`@ActiveProfiles("test")`) and `application-test.properties` provide isolated configuration for tests, typically using an in-memory H2 database. `@TestConfiguration` supplies test-specific beans that replace real implementations.

---

## Resources

- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Mockito Documentation](https://site.mockito.org/)
- [Spring Boot — Testing](https://docs.spring.io/spring-boot/reference/testing/index.html)
- [Spring Framework — @MockitoBean and @MockitoSpyBean](https://docs.spring.io/spring-framework/reference/testing/annotations/integration-spring/annotation-mockitobean.html)
- [Spring MVC Test Framework](https://docs.spring.io/spring-framework/reference/testing/spring-mvc-test-framework.html)
- [Baeldung: Testing in Spring Boot](https://www.baeldung.com/spring-boot-testing)

---

## Lab Assignment: Add Tests to Your Web Application

Write automated tests for your existing application.

**Requirements:**

1. **Write at least 3 unit tests** for a service class using JUnit 5 and Mockito. Mock the repository dependency with `@Mock` and `@InjectMocks`, and verify correct behavior for both success and failure scenarios (e.g., resource found vs. not found).

2. **Write at least 2 repository tests** using `@DataJpaTest` to verify that your custom query methods (derived queries or `@Query` methods) return the correct results. Use `TestEntityManager` to populate test data.

3. **Write at least 2 controller tests** using `@WebMvcTest` and `MockMvc`. Use `@MockitoBean` to mock service dependencies. Test a GET endpoint that returns a page with model attributes, and test a POST endpoint that validates form input — verifying both the success case (redirect) and the validation failure case (form re-displayed with errors).

4. **Configure a test profile** with `application-test.properties` in `src/test/resources/`, using H2 as the test database with `ddl-auto=create-drop`.

5. **Ensure all tests pass** when running `mvn test`.
