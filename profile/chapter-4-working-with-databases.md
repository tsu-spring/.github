# Chapter 4: Working with Databases

---

## 4.1 From In-Memory to Persistent Data

In the previous chapters, we built controllers, rendered templates, handled form submissions, and validated user input. But every time the application restarted, all data was lost — because we were storing it in memory. A real web application needs persistent storage, and that almost always means a database.

This chapter introduces the tools Spring Boot provides for working with relational databases. We will learn how Object-Relational Mapping (ORM) bridges the gap between Java objects and database tables, how to define entities and map relationships between them, how to perform data access through Spring Data JPA repositories, how to write custom queries, and how to implement pagination and sorting. We will also cover transaction management, database configuration for development and production, data initialization, and schema migration tools.

All database access in this chapter uses **Spring Data JPA** — Spring's abstraction layer over the Jakarta Persistence API (JPA), with Hibernate as the underlying ORM implementation.

---

## 4.2 Introduction to ORM and JPA

When you work with a relational database from Java, there is a fundamental mismatch: Java works with objects and class hierarchies, while databases work with tables, rows, and foreign keys. **Object-Relational Mapping (ORM)** is the technique that bridges this gap. Instead of writing raw SQL to insert, update, and query data, you work with Java objects, and the ORM framework translates your operations into the appropriate SQL behind the scenes.

**JPA (Jakarta Persistence API)** is the specification that defines how ORM works in Java. It provides a standard set of annotations and interfaces — `@Entity`, `@Id`, `@Column`, `EntityManager`, and so on — that any compliant implementation must support. **Hibernate** is the most widely used JPA implementation, and it is what Spring Boot uses by default.

**Spring Data JPA** sits on top of JPA and Hibernate. It eliminates boilerplate by generating repository implementations automatically from interface definitions. You declare what data you need; Spring Data JPA figures out how to get it.

### Adding the Dependencies

To use Spring Data JPA, add the starter dependency and a database driver to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- H2 database for development -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

The `spring-boot-starter-data-jpa` dependency brings in Hibernate, the JPA API, Spring Data JPA, and connection pool management (HikariCP). The H2 dependency provides a lightweight, in-memory database that is ideal for development and testing — no installation required, no server to manage.

---

## 4.3 Defining Entities

An **entity** is a Java class that maps to a database table. Each instance of the entity corresponds to a row in the table. Each field in the class maps to a column.

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false)
    private Double price;

    @Column(length = 500)
    private String description;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    // Default constructor required by JPA
    public Product() {}

    // getters and setters
}
```

Let us break down what is happening.

`@Entity` marks the class as a JPA entity — Hibernate will manage it and map it to a database table. Without this annotation, the class is just a regular Java class.

`@Table(name = "products")` specifies the name of the database table. This is optional — if omitted, JPA uses the class name as the table name. Using `@Table` explicitly makes the mapping clear and lets you follow database naming conventions (such as plural, snake_case table names) independently of Java naming conventions.

`@Id` marks the field as the primary key. Every entity must have exactly one `@Id` field.

`@GeneratedValue(strategy = GenerationType.IDENTITY)` tells JPA that the database will generate primary key values automatically using an auto-increment column. This is the most common strategy for MySQL and PostgreSQL.

`@Column` customizes how a field maps to a column. You can specify the column name, whether it allows null values, its maximum length, and whether it must be unique. If you omit `@Column`, JPA will still map the field — it just uses defaults (the field name as the column name, nullable, no length constraint).

The default (no-argument) constructor is required by JPA. Hibernate uses it internally when it creates entity instances from database rows. You can have additional constructors alongside it.

### Key JPA Annotations

| Annotation | Purpose |
|------------|---------|
| `@Entity` | Marks the class as a JPA entity (mapped to a database table). |
| `@Table(name = "...")` | Specifies the table name (optional, defaults to class name). |
| `@Id` | Marks the primary key field. |
| `@GeneratedValue` | Configures auto-generation strategy for the primary key. |
| `@Column` | Customizes column mapping (name, nullable, length, unique, etc.). |
| `@Transient` | Excludes a field from database mapping. |
| `@Enumerated` | Maps an enum field (`EnumType.STRING` or `EnumType.ORDINAL`). |

### Generation Strategies

| Strategy | Behavior |
|----------|----------|
| `GenerationType.IDENTITY` | Database auto-increment column (common with MySQL, PostgreSQL). |
| `GenerationType.SEQUENCE` | Uses a database sequence object (preferred for PostgreSQL in high-throughput scenarios). |
| `GenerationType.AUTO` | Lets the JPA provider choose the strategy based on the database. |
| `GenerationType.TABLE` | Uses a separate table to simulate sequences (rarely used in practice). |

---

## 4.4 Mapping Relationships

Real applications involve entities that relate to each other — a product belongs to a category, a student enrolls in courses, a user has a profile. JPA provides annotations to express these relationships, and Hibernate translates them into the appropriate foreign keys and join tables.

### @ManyToOne and @OneToMany

This is the most common relationship type. Many products belong to one category, and one category has many products:

```java
@Entity
public class Category {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "category", cascade = CascadeType.ALL)
    private List<Product> products = new ArrayList<>();

    // constructors, getters, setters
}

@Entity
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private Double price;

    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;

    // constructors, getters, setters
}
```

Several things are happening here that deserve attention.

`@ManyToOne` on the `Product` side says: "many products can belong to one category." This is the **owning side** of the relationship — the side that contains the foreign key in the database.

`@JoinColumn(name = "category_id")` specifies the name of the foreign key column in the `products` table. If omitted, JPA will generate a default name.

`@OneToMany(mappedBy = "category")` on the `Category` side says: "one category has many products." The `mappedBy` attribute tells JPA that this side does not own the relationship — the `Product` entity owns it through its `category` field. This prevents JPA from creating a redundant join table.

`cascade = CascadeType.ALL` means that operations performed on a `Category` will cascade to its `Product` children. If you save a category, its products are saved too. If you delete a category, its products are deleted too. Use cascading carefully — it is powerful but can lead to unintended mass deletions if applied without thought.

### @ManyToMany

Some relationships are many-to-many — for example, students enrolled in courses, where each student can take multiple courses and each course can have multiple students:

```java
@Entity
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToMany
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

`@JoinTable` configures the join table that the database uses to represent the many-to-many relationship. The `joinColumns` attribute defines the foreign key pointing to the owning side (`Student`), and `inverseJoinColumns` defines the foreign key pointing to the inverse side (`Course`). JPA creates and manages this join table automatically.

### @OneToOne

A one-to-one relationship maps one entity to exactly one instance of another — for example, a user and their profile:

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;

    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "profile_id")
    private UserProfile profile;
}
```

### Cascade Types

| Type | Effect |
|------|--------|
| `CascadeType.PERSIST` | When the parent is saved, the child is saved too. |
| `CascadeType.MERGE` | When the parent is updated, the child is updated too. |
| `CascadeType.REMOVE` | When the parent is deleted, the child is deleted too. |
| `CascadeType.ALL` | All of the above, plus `REFRESH` and `DETACH`. |

### Fetch Types

| Type | Behavior |
|------|----------|
| `FetchType.LAZY` | Related entities are loaded only when you access them (default for collections). |
| `FetchType.EAGER` | Related entities are loaded immediately with the parent (default for single-valued associations). |

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "category_id")
private Category category;
```

`FetchType.LAZY` is almost always the right choice. Eager loading seems convenient but quickly leads to performance problems — loading one entity can trigger a cascade of additional queries that load entire object graphs you never needed. Start with lazy loading and only switch to eager if you have a specific reason.

---

## 4.5 Creating Repositories with Spring Data JPA

This is where Spring Data JPA eliminates an enormous amount of boilerplate. In a traditional JPA application, you would write a data access class with methods that use the `EntityManager` to build queries, execute them, handle transactions, and return results. Spring Data JPA replaces all of that with a single interface declaration:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    // That's it. You now have save(), findById(), findAll(), delete(), count(), and more.
}
```

You do not write an implementation class. Spring Data JPA generates one at runtime. The interface extends `JpaRepository<Product, Long>`, where `Product` is the entity type and `Long` is the type of its primary key.

### JpaRepository vs CrudRepository

| Interface | What It Provides |
|-----------|-----------------|
| `CrudRepository<T, ID>` | Basic CRUD: `save`, `findById`, `findAll`, `delete`, `count`. |
| `JpaRepository<T, ID>` | Everything in `CrudRepository` plus JPA-specific methods: `flush`, `saveAndFlush`, batch deletes, and `findAll(Pageable)` for pagination and sorting. |

In most cases, extend `JpaRepository` — it provides everything `CrudRepository` does and more.

### Using the Repository in a Service

The repository is injected into a service class through constructor injection, just like any other Spring-managed component:

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public List<Product> findAll() {
        return productRepository.findAll();
    }

    public Product findById(Long id) {
        return productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found"));
    }

    public Product save(Product product) {
        return productRepository.save(product);
    }

    public void deleteById(Long id) {
        productRepository.deleteById(id);
    }
}
```

Notice that `findById` returns an `Optional<Product>`. The `orElseThrow` call unwraps it, throwing an exception if no product exists with that ID. This is a safer pattern than returning `null` — it forces you to handle the "not found" case explicitly.

---

## 4.6 Derived Query Methods

One of Spring Data JPA's most distinctive features is the ability to generate queries from method names. You declare a method in your repository interface following a specific naming convention, and Spring Data JPA parses the method name and creates the corresponding query automatically:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    List<Product> findByName(String name);

    List<Product> findByPriceGreaterThan(Double price);

    List<Product> findByNameContainingIgnoreCase(String keyword);

    List<Product> findByCategoryName(String categoryName);

    List<Product> findByPriceBetween(Double minPrice, Double maxPrice);

    Optional<Product> findByNameAndPrice(String name, Double price);

    List<Product> findByNameOrderByPriceAsc(String name);

    int countByCategory(Category category);

    boolean existsByName(String name);
}
```

Each method name is a sentence that Spring Data JPA reads: `findByPriceGreaterThan` means "find entities where the `price` field is greater than the given parameter." `findByNameContainingIgnoreCase` means "find entities where the `name` field contains the given string, ignoring case." The naming convention is precise — Spring Data JPA expects specific keywords in specific positions.

### Common Keywords for Derived Queries

| Keyword | Example | SQL Equivalent |
|---------|---------|----------------|
| `findBy` | `findByName(String name)` | `WHERE name = ?` |
| `Containing` | `findByNameContaining(String s)` | `WHERE name LIKE '%s%'` |
| `GreaterThan` | `findByPriceGreaterThan(Double p)` | `WHERE price > ?` |
| `LessThan` | `findByPriceLessThan(Double p)` | `WHERE price < ?` |
| `Between` | `findByPriceBetween(Double a, Double b)` | `WHERE price BETWEEN ? AND ?` |
| `OrderBy` | `findByNameOrderByPriceAsc(String n)` | `ORDER BY price ASC` |
| `IgnoreCase` | `findByNameIgnoreCase(String n)` | `WHERE LOWER(name) = LOWER(?)` |

Derived query methods work well for simple queries. For anything more complex — queries involving joins, subqueries, aggregations, or conditions that would make the method name unreadable — you should use `@Query` instead.

---

## 4.7 Custom Queries with @Query

When derived query methods become too unwieldy or cannot express the query you need, the `@Query` annotation lets you write JPQL (Java Persistence Query Language) directly:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("SELECT p FROM Product p WHERE p.price > :minPrice ORDER BY p.price ASC")
    List<Product> findExpensiveProducts(@Param("minPrice") Double minPrice);

    @Query("SELECT p FROM Product p WHERE LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    List<Product> searchByName(@Param("keyword") String keyword);

    @Query("SELECT p FROM Product p JOIN p.category c WHERE c.name = :categoryName")
    List<Product> findByCategoryName(@Param("categoryName") String categoryName);

    @Query("SELECT COUNT(p) FROM Product p WHERE p.category.id = :categoryId")
    long countByCategory(@Param("categoryId") Long categoryId);
}
```

JPQL looks similar to SQL but operates on entities and their fields rather than tables and columns. `SELECT p FROM Product p` queries the `Product` entity, not the `products` table. `p.category.name` navigates the relationship defined in the entity class. The `@Param` annotation binds method parameters to named parameters (`:minPrice`, `:keyword`) in the query.

You can also use native SQL when JPQL is not sufficient — for example, when you need database-specific features:

```java
@Query(value = "SELECT * FROM products WHERE price > :minPrice", nativeQuery = true)
List<Product> findExpensiveProductsNative(@Param("minPrice") Double minPrice);
```

Native queries bypass the entity abstraction and work directly with table and column names. Use them sparingly — they tie your code to a specific database dialect and lose the portability that JPQL provides.

---

## 4.8 Pagination and Sorting

When a table contains thousands of rows, loading all of them at once is wasteful and slow. Pagination lets you load data one page at a time, and sorting lets users control the order. Spring Data JPA provides built-in support for both through the `Pageable` interface and `Page<T>` return type.

### In the Repository

Any repository method can accept a `Pageable` parameter and return a `Page`:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    Page<Product> findByCategory(Category category, Pageable pageable);

    Page<Product> findByNameContainingIgnoreCase(String keyword, Pageable pageable);
}
```

### In the Service

The service creates a `PageRequest` (which implements `Pageable`) specifying the page number, page size, and sort order:

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public Page<Product> findAll(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        return productRepository.findAll(pageable);
    }
}
```

`PageRequest.of(page, size, Sort.by("name").ascending())` creates a request for the specified page (zero-indexed), with the given number of items per page, sorted by the `name` field in ascending order. You can chain multiple sort criteria: `Sort.by("category").ascending().and(Sort.by("price").descending())`.

### In the Controller

The controller extracts page and size from request parameters, calls the service, and passes the pagination data to the template:

```java
@GetMapping("/products")
public String listProducts(@RequestParam(defaultValue = "0") int page,
                           @RequestParam(defaultValue = "10") int size,
                           Model model) {
    Page<Product> productPage = productService.findAll(page, size);
    model.addAttribute("products", productPage.getContent());
    model.addAttribute("currentPage", page);
    model.addAttribute("totalPages", productPage.getTotalPages());
    model.addAttribute("totalItems", productPage.getTotalElements());
    return "products/list";
}
```

### Building Paginated Views with FreeMarker

The template iterates over the products and renders navigation controls for moving between pages:

```html
<#import "/spring.ftl" as spring>
<#import "layout.ftlh" as layout>

<@layout.page title="Products">
    <h1>Products</h1>

    <table>
        <thead>
            <tr>
                <th>Name</th>
                <th>Price</th>
                <th>Category</th>
            </tr>
        </thead>
        <tbody>
            <#list products as product>
                <tr>
                    <td>${product.name}</td>
                    <td>${product.price}</td>
                    <td>${product.category.name}</td>
                </tr>
            </#list>
        </tbody>
    </table>

    <!-- Pagination controls -->
    <nav>
        <ul>
            <#if (currentPage > 0)>
                <li>
                    <a href="<@spring.url '/products?page=${currentPage - 1}'/>">Previous</a>
                </li>
            </#if>

            <#list 0..<totalPages as i>
                <li <#if i == currentPage>class="active"</#if>>
                    <a href="<@spring.url '/products?page=${i}'/>">${i + 1}</a>
                </li>
            </#list>

            <#if (currentPage < totalPages - 1)>
                <li>
                    <a href="<@spring.url '/products?page=${currentPage + 1}'/>">Next</a>
                </li>
            </#if>
        </ul>
    </nav>
</@layout.page>
```

The `<#list 0..<totalPages as i>` construct creates a range from 0 up to (but not including) `totalPages`. The current page is highlighted with a CSS class, and the Previous/Next links are conditionally shown only when there is a previous or next page to navigate to.

---

## 4.9 Transaction Management with @Transactional

A **transaction** groups multiple database operations into a single atomic unit — either all of them succeed, or all of them are rolled back. Without transactions, a failure halfway through a multi-step operation could leave your data in an inconsistent state.

Spring provides the `@Transactional` annotation to manage transactions declaratively. You annotate a service method, and Spring wraps its execution in a transaction automatically:

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;

    public OrderService(OrderRepository orderRepository,
                        ProductRepository productRepository) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
    }

    @Transactional
    public Order createOrder(Order order) {
        // If any of these operations fail, all changes are rolled back
        for (OrderItem item : order.getItems()) {
            Product product = productRepository.findById(item.getProductId())
                    .orElseThrow(() -> new RuntimeException("Product not found"));
            product.setStock(product.getStock() - item.getQuantity());
            productRepository.save(product);
        }
        return orderRepository.save(order);
    }
}
```

If any operation inside the `createOrder` method throws an exception, all database changes made within that method are rolled back — the stock quantities revert, the order is not saved. This is exactly the behavior you want: either the entire order is created successfully, or nothing changes.

Key points about `@Transactional`:

Place it on **service methods**, not repository methods. Repository methods are already transactional by default (Spring Data JPA wraps each repository call in its own transaction). The `@Transactional` annotation on a service method is what ties multiple repository calls into a single transaction.

By default, Spring rolls back on **unchecked exceptions** (`RuntimeException` and its subclasses) but not on checked exceptions. If you need rollback on checked exceptions, specify it explicitly: `@Transactional(rollbackFor = Exception.class)`.

For read-only operations, use `@Transactional(readOnly = true)`. This signals to Hibernate that no data will be modified, allowing it to skip dirty-checking optimizations and potentially improve performance.

---

## 4.10 Database Configuration

Spring Boot makes it easy to switch between different databases for different environments. During development, an in-memory H2 database keeps things fast and simple. In production, you use a full database like PostgreSQL or MySQL.

### H2 Configuration (Development)

```properties
# H2 Database (in-memory, for development)
spring.datasource.url=jdbc:h2:mem:devdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# Enable H2 Console
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console

# JPA / Hibernate
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
```

The H2 Console is a web-based database browser accessible at `http://localhost:8080/h2-console`. It lets you inspect your tables, run SQL queries, and verify that your entities are being mapped correctly. This is invaluable during development — when something is not behaving as expected, the console lets you look directly at what is in the database.

`spring.jpa.show-sql=true` logs every SQL statement Hibernate executes to the console. This is useful for understanding what queries your code generates, but should be disabled in production to avoid noise in the logs.

### PostgreSQL Configuration (Production)

To switch to PostgreSQL, add the driver dependency and update your properties:

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.driver-class-name=org.postgresql.Driver
spring.datasource.username=myuser
spring.datasource.password=mypassword

spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
```

Notice the change in `ddl-auto` — from `create-drop` in development to `validate` in production. You never want Hibernate modifying your production schema automatically.

### Hibernate ddl-auto Options

| Value | Behavior |
|-------|----------|
| `none` | No schema management at all. |
| `validate` | Checks that the database schema matches the entities. Throws an exception if they do not match, but never modifies the schema. |
| `update` | Updates the schema to match the entities — adds new columns and tables, but never removes existing ones. |
| `create` | Drops and recreates the schema on every application startup. All existing data is lost. |
| `create-drop` | Same as `create`, but also drops the schema when the application shuts down. Ideal for automated tests. |

For development, `create-drop` is convenient — you start fresh every time. For production, use `validate` (or `none` if you manage the schema externally with a migration tool). Never use `update` in production — it seems safe, but it can produce unexpected schema changes and cannot remove columns or tables.

---

## 4.11 Database Initialization with data.sql and schema.sql

Spring Boot can automatically execute SQL scripts on startup, which is useful for seeding a development database with test data.

`schema.sql` creates tables and is executed before Hibernate initialization (if configured to do so). `data.sql` inserts initial data and runs after the schema is in place.

```sql
-- data.sql
INSERT INTO categories (name) VALUES ('Electronics');
INSERT INTO categories (name) VALUES ('Books');
INSERT INTO categories (name) VALUES ('Clothing');

INSERT INTO products (name, price, category_id) VALUES ('Laptop', 999.99, 1);
INSERT INTO products (name, price, category_id) VALUES ('Spring in Action', 49.99, 2);
```

To ensure that `data.sql` runs after Hibernate has created the schema (which is necessary when using `ddl-auto=create-drop`), add this property:

```properties
spring.jpa.defer-datasource-initialization=true
```

Without this setting, Spring Boot may try to run `data.sql` before the tables exist, resulting in errors.

Place these files in `src/main/resources/` — Spring Boot finds them automatically by convention. This approach works well for development and testing. For production, use a migration tool instead.

---

## 4.12 Programmatic Data Initialization with CommandLineRunner

While `data.sql` works for simple cases, it has limitations. SQL scripts cannot easily generate randomized test data, they do not benefit from your entity model or validation logic, and they quickly become unwieldy when your schema involves relationships. For more complex initialization scenarios, Spring Boot provides a programmatic alternative: the `CommandLineRunner` interface.

A `CommandLineRunner` is a Spring-managed component whose `run` method is executed once, immediately after the application context starts. You can inject repositories into it and use them to populate the database with Java code:

```java
@Component
@RequiredArgsConstructor
public class DataInitializer implements CommandLineRunner {

    private final CategoryRepository categoryRepository;
    private final ProductRepository productRepository;

    @Override
    public void run(String... args) {
        Category electronics = new Category();
        electronics.setName("Electronics");
        categoryRepository.save(electronics);

        Category books = new Category();
        books.setName("Books");
        categoryRepository.save(books);

        Product laptop = new Product();
        laptop.setName("Laptop");
        laptop.setPrice(999.99);
        laptop.setCategory(electronics);
        productRepository.save(laptop);

        Product novel = new Product();
        novel.setName("Spring in Action");
        novel.setPrice(49.99);
        novel.setCategory(books);
        productRepository.save(novel);
    }
}
```

This approach gives you the full power of Java — you can use loops, read from external files (JSON, CSV), generate random data, and enforce the same validation rules your entities define. It is especially useful when your data model involves relationships that would be tedious to express in raw SQL.

### The "Detached Entity Passed to Persist" Pitfall

When populating data programmatically, there is a common and frustrating error that catches many developers off guard. Consider this scenario: you have a `Book` entity with a `@ManyToMany` relationship to `Genre`, and you want to seed books from a JSON file where each book references its genres by name.

Here is the entity setup:

```java
@Entity
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "book_genres",
        joinColumns = @JoinColumn(name = "book_id"),
        inverseJoinColumns = @JoinColumn(name = "genre_id")
    )
    private Set<Genre> genres = new HashSet<>();

    // other fields, constructors, getters, setters
}

@Entity
public class Genre {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToMany(mappedBy = "genres")
    private Set<Book> books = new HashSet<>();
}
```

Now suppose you write a `CommandLineRunner` that first saves the genres, then reads books from JSON and creates `Genre` objects on the fly to attach to each book:

```java
@Component
@RequiredArgsConstructor
public class FakeDataFiller implements CommandLineRunner {

    private final BookRepository bookRepository;
    private final GenreRepository genreRepository;

    @Override
    public void run(String... args) {
        // Step 1: Save genres to the database
        Genre fiction = new Genre();
        fiction.setName("Fiction");
        genreRepository.save(fiction);

        Genre sciFi = new Genre();
        sciFi.setName("Science Fiction");
        genreRepository.save(sciFi);

        // Step 2: Create a book and attach genres — the WRONG way
        Book book = new Book();
        book.setTitle("Dune");

        // Creating a NEW Genre object with the same name, but it is NOT
        // the managed entity we just saved — it is a completely separate
        // Java object that JPA knows nothing about.
        Genre detachedGenre = new Genre();
        detachedGenre.setId(1L);          // Even setting the ID does not help
        detachedGenre.setName("Science Fiction");

        book.setGenres(Set.of(detachedGenre));
        bookRepository.save(book);  // BOOM — Exception!
    }
}
```

This fails with:

```
org.hibernate.PersistentObjectException:
    Detached entity passed to persist: com.example.genre.Genre
```

**What happened?** JPA manages entities through a **persistence context** — a kind of cache that tracks every entity loaded from or saved to the database within the current session. When you called `genreRepository.save(fiction)`, that specific `fiction` object became a **managed** entity in the persistence context.

But the `detachedGenre` object created afterwards is not the same Java object — it is a brand new instance that JPA has never seen. Even though it has the same name and even the same ID, JPA does not recognize it as the entity already in the database. It is a **detached** entity.

When you call `bookRepository.save(book)`, Hibernate sees that `book` is a new entity (no ID yet), so it calls `persist()`. Because the `@ManyToMany` relationship includes `CascadeType.PERSIST`, Hibernate cascades the persist operation to the associated genres. It tries to `persist()` the `detachedGenre` — but that entity already has an ID, which tells Hibernate it is not truly new. Hibernate cannot reconcile this: you are asking it to insert a new entity that already claims to have a database identity. So it throws `PersistentObjectException`.

### The Fix: Always Use Managed Entities

The solution is straightforward — never attach unmanaged entity instances to relationships. Instead, look up the existing entities from the repository so you get back the managed references:

```java
@Component
@RequiredArgsConstructor
public class FakeDataFiller implements CommandLineRunner {

    private final BookRepository bookRepository;
    private final GenreRepository genreRepository;

    @Override
    public void run(String... args) {
        // Step 1: Save genres
        Genre fiction = new Genre();
        fiction.setName("Fiction");
        genreRepository.save(fiction);

        Genre sciFi = new Genre();
        sciFi.setName("Science Fiction");
        genreRepository.save(sciFi);

        // Step 2: Create a book and attach genres — the RIGHT way
        Book book = new Book();
        book.setTitle("Dune");

        // Look up the genre from the repository — this returns a MANAGED entity
        Genre managedSciFi = genreRepository.findByName("Science Fiction");

        book.setGenres(Set.of(managedSciFi));
        bookRepository.save(book);  // Works correctly
    }
}
```

The `genreRepository.findByName("Science Fiction")` call returns the same managed entity that the persistence context is already tracking. When Hibernate cascades the persist to this genre, it recognizes it as an existing managed entity and simply creates the join table entry — no attempt to insert a duplicate.

This pattern applies to any relationship where the referenced entity already exists in the database. Whether you are working with `@ManyToOne`, `@ManyToMany`, or `@OneToOne`, the rule is the same: **if the entity is already persisted, fetch it from the repository before attaching it to another entity**. Creating a new Java object with the same ID or same field values is not enough — JPA cares about object identity within the persistence context, not just database identity.

---

## 4.13 Schema Migration with Flyway and Liquibase (Overview)

Manually managing database schema changes is risky — it is easy to forget a step, apply changes out of order, or lose track of what has been applied to which environment. Schema migration tools solve this by providing version-controlled, repeatable, and automated database migrations.

### Flyway

Flyway uses numbered SQL scripts that are applied in order. Each script is named with a version prefix — `V1__`, `V2__`, and so on — and Flyway tracks which scripts have already been executed. It only runs new ones.

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

Place migration scripts in `src/main/resources/db/migration/`:

```sql
-- V1__Create_products_table.sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    description VARCHAR(500)
);
```

```sql
-- V2__Add_category_column.sql
ALTER TABLE products ADD COLUMN category_id BIGINT REFERENCES categories(id);
```

Flyway creates a metadata table (`flyway_schema_history`) in your database that records which migrations have been applied and when. On each application startup, it compares the migration files on disk to the metadata table and applies only the ones that are new. This makes deployments safe and repeatable — you can be confident that every environment has the same schema.

### Liquibase

Liquibase takes a different approach — it supports XML, YAML, JSON, or SQL changesets and provides more flexibility in how migrations are defined. It is more powerful than Flyway for complex scenarios but also more complex to set up.

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

Both tools are production-grade and widely used. Flyway is simpler and a good default choice. Liquibase is worth considering if you need rollback support, multi-database targeting, or more granular control over migration logic.

---

## 4.14 Putting It All Together: A Complete Example

To see how entities, repositories, services, and controllers connect in a real application, here is a complete example — a "Gossip" posting application:

### The Entity

```java
@Entity
@Table(name = "gossips")
public class Gossip {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Size(max = 280)
    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private User author;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    // constructors, getters, setters
}
```

Notice that the entity combines JPA annotations (`@Entity`, `@ManyToOne`, `@JoinColumn`) with Bean Validation annotations (`@NotBlank`, `@Size`) from Chapter 3. This is a common pattern — the same class serves both as a database mapping and as a form-backing object with validation rules.

### The Repository

```java
public interface GossipRepository extends JpaRepository<Gossip, Long> {

    List<Gossip> findByAuthorOrderByCreatedAtDesc(User author);

    Page<Gossip> findAllByOrderByCreatedAtDesc(Pageable pageable);

    @Query("SELECT g FROM Gossip g WHERE g.content LIKE %:keyword%")
    List<Gossip> search(@Param("keyword") String keyword);
}
```

This repository uses derived query methods for common access patterns and a `@Query` method for search functionality that would be awkward to express through method naming alone.

### The Service

```java
@Service
public class GossipService {

    private final GossipRepository gossipRepository;

    public GossipService(GossipRepository gossipRepository) {
        this.gossipRepository = gossipRepository;
    }

    public Page<Gossip> getLatestGossips(int page, int size) {
        return gossipRepository.findAllByOrderByCreatedAtDesc(
                PageRequest.of(page, size));
    }
}
```

The service is thin by design — it delegates to the repository and provides a clean API for the controller to use. As the application grows, the service layer is where business logic (beyond simple CRUD) will live.

---

## Summary

This chapter covered the tools and techniques for connecting a Spring Boot application to a relational database.

**ORM and JPA** bridge the gap between Java objects and database tables. Hibernate is the JPA implementation that Spring Boot uses by default, and Spring Data JPA provides a higher-level abstraction that eliminates most boilerplate data access code.

**Entities** are Java classes annotated with `@Entity`, `@Id`, `@GeneratedValue`, and `@Column` that map to database tables. Each entity instance corresponds to a row.

**Relationships** between entities are expressed with `@ManyToOne` / `@OneToMany`, `@ManyToMany`, and `@OneToOne`. The owning side of the relationship holds the `@JoinColumn`, and `mappedBy` on the inverse side prevents duplicate foreign keys.

**Repositories** extend `JpaRepository` and provide CRUD operations, pagination, and sorting without any implementation code. **Derived query methods** generate queries from method names, and `@Query` allows you to write custom JPQL or native SQL.

**Pagination and sorting** are handled through `Pageable`, `PageRequest`, and `Page<T>`, allowing you to load data one page at a time and present navigation controls in your templates.

**`@Transactional`** groups multiple database operations into an atomic unit — either all succeed or all are rolled back. Place it on service methods to ensure data consistency.

**Database configuration** uses `application.properties` to switch between H2 for development and PostgreSQL or MySQL for production. The `ddl-auto` property controls how Hibernate manages the schema.

**Database initialization** with `data.sql` seeds development databases with simple test data. For more complex scenarios, **`CommandLineRunner`** provides programmatic initialization with full access to your repositories and entity model. When working with relationships during initialization, always fetch existing entities from the repository rather than creating new Java objects — otherwise, JPA will throw a "detached entity passed to persist" error. **Schema migration** tools like Flyway and Liquibase provide version-controlled, repeatable migrations for production environments.

---

## Resources

- [Spring Data JPA Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Jakarta Persistence (JPA) Specification](https://jakarta.ee/specifications/persistence/)
- [Hibernate ORM Documentation](https://hibernate.org/orm/documentation/)
- [Baeldung: Spring Data JPA](https://www.baeldung.com/the-persistence-layer-with-spring-data-jpa)
- [H2 Database](https://www.h2database.com/)
- [Flyway Documentation](https://documentation.red-gate.com/fd)
- [Liquibase Documentation](https://docs.liquibase.com/)

---

## Lab Assignment: Build a Database-Backed Web Application

Extend your interactive web application from Chapter 3 with persistent data storage.

**Requirements:**

1. **Define JPA entities** with proper annotations (`@Entity`, `@Id`, `@GeneratedValue`, `@Column`). Your entities should include validation annotations from Chapter 3 so they serve as both database mappings and form-backing objects.

2. **Map at least one relationship** between entities (e.g., `@ManyToOne` / `@OneToMany` between blog posts and a category or author entity).

3. **Create repository interfaces** extending `JpaRepository` and add at least two derived query methods or `@Query` methods — for example, a search method and a method that filters by category.

4. **Implement pagination** for listing pages — display a configurable number of items per page with Previous/Next navigation controls rendered in FreeMarker.

5. **Use H2 database** for development with the H2 console enabled, and pre-populate data using `data.sql`.

6. **Add `@Transactional`** where appropriate — for example, on service methods that modify multiple entities in a single operation.
