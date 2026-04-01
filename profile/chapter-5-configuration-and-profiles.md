# Chapter 5: Configuration and Profiles

---

## 5.1 From Hardcoded to Configurable

In the previous chapters, we built controllers, templates, entities, and repositories — but every time we needed to change a value, we edited Java code. The database URL lived in `application.properties` because Spring Boot told us to put it there, but we never stopped to think about configuration as a concept in its own right. What happens when you need to deploy the same application to a test server with a different database? Or change a timeout value without recompiling? Or hide production credentials from your source code?

This chapter introduces Spring Boot's externalized configuration system — the mechanisms that let you separate configuration from code so that the same compiled application can behave differently in different environments. We will learn how Spring Boot finds and prioritizes configuration values, how to inject simple values with `@Value`, how to bind structured configuration to type-safe Java objects with `@ConfigurationProperties`, and how to use Spring profiles to maintain separate configurations for development, testing, and production. By the end, you will be able to switch your application between H2 and PostgreSQL by changing a single profile — without touching any Java code.

---

## 5.2 Externalized Configuration

The core idea of externalized configuration is simple: values that might change between environments should not be baked into your code. Instead, they live in external sources that Spring Boot reads at startup and makes available to your application.

Spring Boot supports several types of configuration sources, including `application.properties` and `application.yml` files, operating system environment variables, Java system properties, and command-line arguments. All of these sources feed into a single, unified `Environment` abstraction. Your code asks the `Environment` for a value, and Spring Boot figures out where it came from.

Configuration values can be consumed in three ways: injected directly into bean fields using the `@Value` annotation, accessed programmatically through Spring's `Environment` object, or bound to structured Java objects using the `@ConfigurationProperties` annotation. Each approach has its place, and we will cover them in order of increasing sophistication.

---

## 5.3 Configuration File Formats

Spring Boot supports two file formats for configuration: **properties** and **YAML**. Both live in `src/main/resources/` and are loaded automatically at startup.

### application.properties

The properties format uses flat key-value pairs, one per line:

```properties
app.title=My Spring Boot App
app.max-connections=120
app.servers=dev,qa,prod
server.port=8080
```

### application.yml

The YAML format uses indentation to express hierarchy:

```yaml
app:
  title: My Spring Boot App
  max-connections: 120
  servers:
    - dev
    - qa
    - prod
server:
  port: 8080
```

Both formats are interchangeable — Spring Boot treats them identically. YAML tends to be more readable for deeply nested configuration because it avoids repeating the prefix on every line. Properties files are simpler for flat key-value pairs. Choose whichever your team prefers and be consistent. You can even use both in the same project, though there is rarely a reason to.

---

## 5.4 How Spring Boot Resolves Configuration Values

When the same property is defined in multiple sources, Spring Boot needs a way to decide which value wins. It uses a well-defined priority order. The following is a simplified version of the actual resolution order, from lowest to highest priority:

1. Default values (defined in code with `@Value("${key:default}")`)
2. `application.properties` or `application.yml` in the classpath
3. Profile-specific configuration files (`application-dev.properties`, etc.)
4. OS environment variables
5. Java system properties (`-Dkey=value`)
6. Command-line arguments (`--key=value`)

Higher-priority sources override lower-priority ones. This means a command-line argument always beats a properties file, and an environment variable always beats the default packaged in the JAR. This design is deliberate — it lets you package sensible defaults in your configuration files and override specific values at deploy time without modifying the artifact.

For the full, exhaustive ordering (which includes several additional sources), see the [Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html) section of the Spring Boot reference documentation.

---

## 5.5 Simple Property Injection with @Value

The most direct way to read a configuration value is the `@Value` annotation. You place it on a field in any Spring-managed component, and Spring injects the value at startup:

```java
@Service
public class AppInfoService {

    @Value("${app.title}")
    private String title;

    @Value("${app.max-connections}")
    private int maxConnections;

    @Value("${app.servers}")
    private List<String> servers;

    @Value("${feature.enabled:true}")
    private boolean featureEnabled;
}
```

The expression `${app.title}` tells Spring to look up the property `app.title` from the environment. Spring handles type conversion automatically — it converts the string `"120"` to an `int` for `maxConnections`, and splits the comma-separated string `"dev,qa,prod"` into a `List<String>` for `servers`.

The colon syntax in `${feature.enabled:true}` provides a default value. If `feature.enabled` is not defined anywhere, the field will be set to `true`. Without a default, a missing property will cause the application to fail at startup with an error — which is actually desirable if the value is mandatory.

### When @Value Is Enough — and When It Is Not

`@Value` works well for injecting one or two values into a single class. But it has real limitations that become apparent as your application grows.

**Scattered configuration.** If you use `@Value("${app.title}")` in three different classes, the property name `app.title` appears in three places. Rename the property in your configuration file and you must find and update every `@Value` annotation that references it — the compiler will not help you.

**No type safety at the binding level.** Spring converts the string value to the declared field type, but it does not validate the result. If `app.max-connections` contains a negative number, `@Value` will happily inject it. You only discover the problem when something breaks at runtime.

**No IDE support for navigation.** Your IDE cannot easily show you all the places where a particular property is used, or autocomplete property names inside `@Value` expressions.

For anything beyond a few simple values, `@ConfigurationProperties` is a better choice.

---

## 5.6 Structured Configuration with @ConfigurationProperties

The `@ConfigurationProperties` annotation takes a different approach: instead of injecting individual values into scattered fields, it binds an entire group of properties to a dedicated Java class. The class becomes a single, type-safe representation of a configuration section.

### Defining a Configuration Properties Class

```java
@Component
@ConfigurationProperties(prefix = "my.app")
public class MyAppProperties {

    private String name;
    private int timeout;
    private List<String> servers;

    // getters and setters

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public int getTimeout() { return timeout; }
    public void setTimeout(int timeout) { this.timeout = timeout; }

    public List<String> getServers() { return servers; }
    public void setServers(List<String> servers) { this.servers = servers; }
}
```

### The Corresponding Configuration

```yaml
my:
  app:
    name: ExampleApp
    timeout: 30
    servers:
      - dev
      - test
      - prod
```

The `prefix = "my.app"` attribute tells Spring Boot to look for properties that start with `my.app` and bind them to the class fields by name. `my.app.name` maps to the `name` field, `my.app.timeout` maps to `timeout`, and `my.app.servers` maps to `servers`. Spring Boot handles nested objects, lists, maps, and complex types automatically.

The `@Component` annotation registers the class as a Spring bean so it can be injected into other components through constructor injection:

```java
@Service
public class GreetingService {

    private final MyAppProperties appProperties;

    public GreetingService(MyAppProperties appProperties) {
        this.appProperties = appProperties;
    }

    public String getWelcomeMessage() {
        return "Welcome to " + appProperties.getName()
             + " (timeout: " + appProperties.getTimeout() + "s)";
    }
}
```

### Why This Is Better Than @Value

All configuration for the `my.app` prefix lives in one class. If you rename a property, you change it in one place. Your IDE can autocomplete the getter methods. The class is easy to test — you can create an instance and set values directly without needing the Spring context. And because it is a regular Java object, you can add validation to it.

### Validating Configuration Properties

You can apply Bean Validation annotations (the same ones we used for form validation in Chapter 3) to a `@ConfigurationProperties` class. Add the `@Validated` annotation to the class, and Spring Boot will validate the bound values at startup:

```java
@Validated
@Component
@ConfigurationProperties(prefix = "my.app")
public class MyAppProperties {

    @NotBlank
    private String name;

    @Min(1)
    private int timeout;

    private List<String> servers;

    // getters and setters
}
```

If `my.app.name` is blank or `my.app.timeout` is less than 1, the application will fail to start with a clear error message. This is exactly what you want — configuration problems should be caught immediately at startup, not hours later when a request hits a code path that reads the bad value.

This requires the validation starter (which you likely already have from Chapter 3):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

---

## 5.7 Loading Configuration from Custom Files

By default, Spring Boot reads `application.properties` (or `application.yml`) from the classpath root. But sometimes you want to organize configuration into separate files — keeping payment gateway settings in one file, email settings in another, and so on.

The `@PropertySource` annotation lets you load additional property files:

```java
@Configuration
@PropertySource("classpath:email-config.properties")
public class EmailConfig {

    @Value("${email.smtp.host}")
    private String smtpHost;

    @Value("${email.smtp.port}")
    private int smtpPort;
}
```

There is an important limitation: `@PropertySource` only supports `.properties` files by default. It does not natively support YAML. If you need to load a YAML file, you must provide a custom `PropertySourceFactory`:

```java
public class YamlPropertySourceFactory implements PropertySourceFactory {

    @Override
    public PropertySource<?> createPropertySource(String name, EncodedResource resource)
            throws IOException {
        YamlPropertiesFactoryBean factory = new YamlPropertiesFactoryBean();
        factory.setResources(resource.getResource());
        return new PropertiesPropertySource(
                resource.getResource().getFilename(), factory.getObject());
    }
}
```

Then reference the factory in the annotation:

```java
@Configuration
@PropertySource(value = "classpath:email-config.yml", factory = YamlPropertySourceFactory.class)
public class EmailConfig {
    // ...
}
```

In practice, most applications keep their configuration in `application.properties` or `application.yml` and use profile-specific files (covered in the next section) to separate environment-specific values. Custom property source files are useful when you have a large amount of configuration that is logically separate from the rest of the application — for example, configuration for an external integration that has its own lifecycle.

---

## 5.8 Spring Profiles

Up to this point, we have one set of configuration values that applies everywhere. But a real application runs in multiple environments — a developer's laptop, a CI server, a staging environment, production — and each environment needs different settings. The database URL, the log level, the external service endpoints, and the feature flags may all differ.

**Spring profiles** solve this by letting you define named groups of configuration and activate only the relevant group for a given environment. A profile is just a label — `dev`, `test`, `prod`, `staging`, or any name you choose. Spring Boot loads the configuration associated with the active profile and ignores everything else.

### Activating a Profile

Profiles can be activated in several ways, each suited to a different scenario:

**In application.properties** (useful for setting a default during development):

```properties
spring.profiles.active=dev
```

**As a command-line argument** (useful for deployment scripts):

```bash
java -jar myapp.jar --spring.profiles.active=prod
```

**As an environment variable** (useful for container-based deployments):

```bash
export SPRING_PROFILES_ACTIVE=prod
```

**Programmatically** (rarely needed, but available):

```java
SpringApplication app = new SpringApplication(MyApplication.class);
app.setAdditionalProfiles("test");
app.run(args);
```

The command-line argument and environment variable approaches are the most common in practice. They let you deploy the exact same JAR to every environment and control behavior entirely from outside the application.

---

## 5.9 Profile-Specific Configuration Files

Spring Boot uses a naming convention to associate configuration files with profiles. A file named `application-{profile}.properties` or `application-{profile}.yml` is loaded only when that profile is active.

For example, given these files:

```
src/main/resources/
├── application.properties          ← always loaded
├── application-dev.properties      ← loaded when "dev" profile is active
└── application-prod.properties     ← loaded when "prod" profile is active
```

`application.properties` contains settings that are common to all environments. `application-dev.properties` and `application-prod.properties` contain environment-specific overrides. Values in the profile-specific file take precedence over values in the base file.

**application.properties** (shared defaults):

```properties
app.title=My Application
logging.level.root=INFO
```

**application-dev.properties:**

```properties
server.port=8081
logging.level.root=DEBUG
```

**application-prod.properties:**

```properties
server.port=80
logging.level.root=WARN
```

When the `dev` profile is active, the server runs on port 8081 with DEBUG logging. When `prod` is active, it runs on port 80 with WARN logging. The `app.title` comes from the base file in both cases because neither profile overrides it.

### Multi-Document Files

If you prefer not to manage multiple files, YAML supports combining profile-specific configurations into a single file using the `---` document separator:

```yaml
# Common configuration
server:
  port: 8080
---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081
---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

Each section separated by `---` is a separate document. The `spring.config.activate.on-profile` property tells Spring Boot which profile the section applies to. The first section (with no profile activation) provides defaults. This approach works well for small applications but can become hard to navigate as the configuration grows — at that point, separate files are clearer.

---

## 5.10 Conditional Bean Registration with @Profile

Configuration files control property values. But sometimes you need to swap entire components — a different `DataSource` implementation, a different email sender, a mock service for testing. The `@Profile` annotation makes beans conditional on the active profile:

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource h2DataSource() {
        return new HikariDataSource(new HikariConfig("/h2.properties"));
    }

    @Bean
    @Profile("prod")
    public DataSource postgresDataSource() {
        return new HikariDataSource(new HikariConfig("/postgres.properties"));
    }
}
```

When the `dev` profile is active, Spring creates the H2 data source and ignores the PostgreSQL bean definition. When `prod` is active, the reverse happens. The rest of the application — services, controllers, repositories — does not change. They inject a `DataSource` and do not care which implementation they get.

`@Profile` can also be placed on `@Configuration` classes to conditionally register entire groups of beans:

```java
@Configuration
@Profile("dev")
public class DevConfig {
    // All beans in this class are registered only in the "dev" profile
}
```

Use `@Profile` judiciously. It is powerful for swapping infrastructure components (data sources, message brokers, caching strategies), but overusing it leads to an application that is hard to reason about — you can no longer look at the code and know which beans are active without also checking the profile. If you find yourself applying `@Profile` to many classes, consider whether profile-specific property files alone would achieve the same effect.

---

## 5.11 Profile Groups

Sometimes a single environment requires activating multiple profiles at once. For example, a production deployment might need both `proddb` (for the production database) and `prodsecurity` (for production security settings). Rather than requiring operators to remember all the profiles, you can define a group:

```properties
spring.profiles.group.production=proddb,prodsecurity
```

Now activating the `production` profile automatically activates `proddb` and `prodsecurity` as well. This keeps the activation simple (`--spring.profiles.active=production`) while allowing fine-grained separation of configuration concerns.

---

## 5.12 Environment Variables and Command-Line Arguments

Beyond property files, Spring Boot reads configuration from the operating system environment and from command-line arguments. This is essential for production deployments where you do not want sensitive values — database credentials, API keys, encryption secrets — committed to source control.

### Environment Variables

Spring Boot automatically maps environment variables to property names using a relaxed binding convention: uppercase the name, replace dots with underscores, and replace hyphens with underscores.

```bash
export MY_APP_NAME=ProductionApp
export MY_APP_TIMEOUT=60
export SERVER_PORT=9090
```

These map to `my.app.name`, `my.app.timeout`, and `server.port` respectively. If you also have these properties defined in `application.properties`, the environment variables take precedence.

### Command-Line Arguments

Properties can be passed directly when starting the application:

```bash
java -jar myapp.jar --server.port=9090 --my.app.name=CLIApp
```

Command-line arguments have higher priority than both property files and environment variables, making them useful for quick overrides during development or for deployment scripts that need to set specific values.

### Property Placeholders

Configuration values can reference other properties — including environment variables — using the `${...}` placeholder syntax:

```yaml
spring:
  datasource:
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

This is a best practice for credentials: the `application-prod.yml` file lives in source control and defines the structure, but the actual secret values come from environment variables that are set on the deployment server or injected by a secrets manager.

---

## 5.13 Practical Use Case: Switching Between H2 and PostgreSQL

This is the scenario that ties the chapter together. During development, you want an embedded H2 database that requires no setup. In production, you want PostgreSQL. Profiles make this seamless.

**application.properties** (set the default profile for development):

```properties
spring.profiles.active=dev
```

**application-dev.yml:**

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:devdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: true
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
```

**application-prod.yml:**

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/proddb
    driver-class-name: org.postgresql.Driver
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
```

During development, you run the application normally. The `dev` profile is active by default, H2 starts in memory, the H2 console is available for inspection, and Hibernate recreates the schema on every restart.

For production deployment:

```bash
export SPRING_PROFILES_ACTIVE=prod
export DB_USERNAME=myuser
export DB_PASSWORD=mysecretpassword
java -jar myapp.jar
```

The production profile activates PostgreSQL, disables SQL logging, and sets `ddl-auto` to `validate` so Hibernate checks the schema but never modifies it. The credentials come from environment variables, not from a file in the repository.

No Java code changed. No recompilation. The same JAR runs in both environments — the only difference is which profile is active and which environment variables are set.

---

## 5.14 Best Practices

A few guidelines that will save you trouble as your configuration grows:

**Keep profiles simple.** Two or three profiles (`dev`, `test`, `prod`) cover most applications. Every additional profile is another combination to test. If you find yourself creating profiles like `dev-mysql-verbose-with-caching`, your configuration model is too complex.

**Never store credentials in property files.** Database passwords, API keys, and encryption secrets should come from environment variables, a secrets manager, or a vault — not from files that get committed to source control. Even `application-prod.properties` should use placeholders like `${DB_PASSWORD}`, not literal values.

**Set a default profile.** If no profile is explicitly activated, Spring Boot runs with no profile at all, which means none of your profile-specific configuration files are loaded. Set `spring.profiles.active=dev` in your base `application.properties` so the application has sensible defaults out of the box.

**Document your profiles.** A new developer joining the team should be able to look at your project and understand which profiles exist, what each one does, and how to activate them. A section in your README is usually sufficient.

**Prefer @ConfigurationProperties over @Value.** For anything more than one or two values, `@ConfigurationProperties` provides type safety, validation, IDE support, and a single point of change. Reserve `@Value` for quick, isolated cases.

**Validate early.** Add `@Validated` and Bean Validation annotations to your `@ConfigurationProperties` classes. It is far better to fail at startup with a clear message ("my.app.timeout must be at least 1") than to discover the problem in production when a request triggers the broken code path.

---

## Summary

This chapter covered Spring Boot's externalized configuration system — the tools that separate configuration from code so the same application can run unchanged across different environments.

**Configuration sources** include property files, YAML files, environment variables, and command-line arguments. Spring Boot resolves them in a defined priority order, with command-line arguments at the top and file-based defaults at the bottom.

**`@Value`** injects individual property values into bean fields. It supports default values and type conversion, but becomes unwieldy when configuration is spread across many classes.

**`@ConfigurationProperties`** binds an entire group of properties to a type-safe Java class. Combined with `@Validated` and Bean Validation annotations, it catches configuration errors at startup rather than at runtime.

**`@PropertySource`** loads additional property files beyond the default `application.properties`, useful for organizing configuration into logical groups.

**Spring profiles** associate configuration with named environments — `dev`, `test`, `prod`. Profile-specific files (`application-dev.properties`) override the base configuration when the corresponding profile is active.

**`@Profile`** on bean definitions makes entire components conditional on the active profile, enabling you to swap infrastructure implementations (like databases) without changing application code.

**Environment variables and command-line arguments** provide the highest-priority configuration sources, ideal for injecting credentials and deployment-specific overrides without committing sensitive values to source control.

---

## Resources

- [Externalized Configuration — Spring Boot](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Profiles — Spring Boot](https://docs.spring.io/spring-boot/reference/features/profiles.html)
- [Type-Safe Configuration Properties — Spring Boot](https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config.typesafe-configuration-properties)
- [YAML Configuration — Spring Boot](https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config.yaml)

---

## Lab Assignment: Configure Your Application with Profiles

Apply externalized configuration and profiles to your web application.

**Requirements:**

1. **Externalize application-specific configuration.** Identify values in your application that are currently hardcoded or scattered across classes — application name, page sizes, feature flags, or similar — and move them into a configuration file. Define a `@ConfigurationProperties` class with a meaningful prefix that reads these values, validates them with `@Validated` and Bean Validation annotations, and provides them to the rest of the application through constructor injection.

2. **Define two profiles — `dev` and `prod`.** The development profile should use an H2 in-memory database with the H2 console enabled, `ddl-auto` set to `create-drop`, and SQL logging turned on. The production profile should use PostgreSQL (or another production database), `ddl-auto` set to `validate`, SQL logging disabled, and credentials supplied through environment variables rather than hardcoded in the file.

3. **Verify profile switching.** Confirm that activating the `dev` profile starts the application with H2 and that activating `prod` connects to PostgreSQL — without modifying any Java code. Test activation via `application.properties`, via command-line argument (`--spring.profiles.active=prod`), and via environment variable (`SPRING_PROFILES_ACTIVE=prod`).
