# Chapter 8: Securing Web Applications

---

## 8.1 Why Security Matters

Every application we have built so far has been completely open — anyone can access any page, submit any form, call any API endpoint, and modify any data. In a development environment, this is convenient. In production, it is unacceptable. Web applications are exposed to the internet and are constantly targeted by attackers.

This chapter introduces Spring Security — the framework that handles authentication (who are you?) and authorization (what are you allowed to do?) for Spring Boot applications. We will learn how to add security to a project, configure access rules, implement different strategies for storing user credentials, encode passwords safely, enforce role-based access control, customize login and logout pages, protect REST APIs, configure CORS, and understand the difference between session-based and token-based authentication.

Spring Boot 4 ships with **Spring Security 7**, which removes several deprecated APIs from earlier versions and changes some defaults. All code in this chapter uses the current API.

---

## 8.2 Common Web Vulnerabilities

Before diving into Spring Security's features, it helps to understand the threats it protects against:

**CSRF (Cross-Site Request Forgery)** — An attacker tricks a logged-in user into submitting a malicious request. For example, a hidden form on an attacker's website could transfer money from the victim's bank account if the bank application does not verify the origin of the request. Spring Security includes CSRF protection by default.

**XSS (Cross-Site Scripting)** — An attacker injects malicious JavaScript into a page viewed by other users. If user input is rendered without escaping, the injected script can steal cookies, redirect users, or modify page content. FreeMarker's default output escaping (configured in Chapter 3) helps prevent this.

**SQL Injection** — An attacker injects malicious SQL through user input to read, modify, or delete database data. Using JPA with parameterized queries (as we did in Chapter 4) prevents this by separating SQL structure from data.

**Broken Authentication** — Weak passwords, missing logout functionality, session fixation, or predictable session IDs allow attackers to impersonate legitimate users.

**Insecure Direct Object References** — Users can access resources by manipulating IDs in URLs — changing `/orders/123` to `/orders/124` to view another user's order.

Spring Security provides defenses against many of these out of the box. Understanding the vulnerabilities helps you appreciate why certain security features exist and why disabling them (particularly CSRF protection) should never be done without careful thought.

---

## 8.3 Adding Spring Security to a Project

Add the Spring Security starter dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

The moment you add this dependency and restart your application, everything changes. Spring Security automatically secures **all endpoints** — every page and every API call requires authentication. It generates a default user named `user` with a random password printed to the console at startup. It enables a default login form at `/login`. It enables CSRF protection. It adds security response headers (`X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`).

This "secure by default" philosophy is deliberate. It is far safer to start with everything locked down and selectively open access than to start open and try to remember to lock things down later.

---

## 8.4 Configuring the Security Filter Chain

The default configuration — lock everything and provide a random password — is not useful for a real application. To customize security behavior, you create a configuration class with a `SecurityFilterChain` bean:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/home", "/register", "/css/**", "/js/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
                .permitAll()
            );

        return http.build();
    }
}
```

This configuration defines three layers of rules.

The **authorization rules** (`authorizeHttpRequests`) specify who can access what. Rules are evaluated in order — the first match wins. Public pages (`/`, `/home`, `/register`) and static resources (`/css/**`, `/js/**`) are accessible to everyone. Admin pages require the `ADMIN` role. User pages require either `USER` or `ADMIN`. Everything else requires authentication but no specific role.

The **form login** configuration tells Spring Security to use a custom login page at `/login` (which we will create as a FreeMarker template) and redirect users to `/dashboard` after a successful login. The `permitAll()` call ensures that the login page itself is accessible without authentication — without it, unauthenticated users would be redirected to the login page, which would redirect them to the login page, in an infinite loop.

The **logout** configuration specifies the logout URL and where to redirect after logout. Spring Security handles the actual logout — invalidating the session, clearing the security context, and deleting cookies.

### Key Authorization Methods

| Method | Access Rule |
|--------|-------------|
| `permitAll()` | Allow access without authentication. |
| `authenticated()` | Require authentication (any role). |
| `hasRole("ADMIN")` | Require the `ADMIN` role. |
| `hasAnyRole("USER", "ADMIN")` | Require any of the listed roles. |
| `hasAuthority("SCOPE_read")` | Require a specific authority (more granular than roles). |

The difference between roles and authorities is subtle but important. A role is a high-level concept (`ADMIN`, `USER`). An authority is a fine-grained permission (`SCOPE_read`, `product:write`). Internally, Spring Security stores roles as authorities with a `ROLE_` prefix — `hasRole("ADMIN")` checks for the authority `ROLE_ADMIN`. For most applications, roles are sufficient.

---

## 8.5 Authentication: In-Memory Users

For development, testing, or very simple applications, you can define users directly in your configuration:

```java
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.builder()
            .username("user")
            .password(passwordEncoder().encode("password"))
            .roles("USER")
            .build();

    UserDetails admin = User.builder()
            .username("admin")
            .password(passwordEncoder().encode("admin123"))
            .roles("USER", "ADMIN")
            .build();

    return new InMemoryUserDetailsManager(user, admin);
}

@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

`InMemoryUserDetailsManager` stores users in memory — they exist only while the application is running. This is useful for quick testing but is not suitable for production, where you need users stored in a database with the ability to register, change passwords, and manage accounts.

Notice that passwords are encoded with `passwordEncoder().encode(...)`. Spring Security refuses to work with plain-text passwords. We will discuss password encoding in detail shortly.

---

## 8.6 Authentication: JDBC-Based Users

Spring Security can authenticate users directly from database tables using `JdbcUserDetailsManager`:

```java
@Bean
public UserDetailsService userDetailsService(DataSource dataSource) {
    return new JdbcUserDetailsManager(dataSource);
}
```

This expects a specific table structure:

```sql
CREATE TABLE users (
    username VARCHAR(50) NOT NULL PRIMARY KEY,
    password VARCHAR(100) NOT NULL,
    enabled BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE authorities (
    username VARCHAR(50) NOT NULL,
    authority VARCHAR(50) NOT NULL,
    FOREIGN KEY (username) REFERENCES users(username)
);
```

`JdbcUserDetailsManager` handles authentication, user creation, password changes, and authority management. It works well when you are comfortable with its predefined table schema. If you need a different schema — for example, using an `id` column instead of `username` as the primary key, or storing roles differently — you need a custom `UserDetailsService`.

---

## 8.7 Authentication: Custom UserDetailsService

For full control over how users are loaded, implement the `UserDetailsService` interface:

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        AppUser appUser = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException(
                        "User not found: " + username));

        return User.builder()
                .username(appUser.getUsername())
                .password(appUser.getPassword())
                .roles(appUser.getRoles().toArray(new String[0]))
                .build();
    }
}
```

This single method — `loadUserByUsername` — is the only thing Spring Security needs to authenticate users. It looks up the user in your database, and if found, returns a `UserDetails` object containing the username, encoded password, and roles. Spring Security then compares the password from the login form (after encoding) with the stored password.

### The User Entity

```java
@Entity
@Table(name = "app_users")
public class AppUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "user_roles", joinColumns = @JoinColumn(name = "user_id"))
    @Column(name = "role")
    private Set<String> roles = new HashSet<>();

    // constructors, getters, setters
}
```

The `@ElementCollection` annotation maps the roles as a collection of simple values in a separate table (`user_roles`). Each row in that table contains a `user_id` and a `role` string. `FetchType.EAGER` ensures that roles are loaded immediately with the user — this is necessary because Spring Security needs the roles during authentication, which happens outside the normal JPA session.

The table name `app_users` avoids conflicts with `users`, which is a reserved word in some databases (notably PostgreSQL).

When Spring Boot detects a `UserDetailsService` bean in the application context, it automatically uses it for authentication — you do not need to explicitly wire it into the security configuration.

---

## 8.8 Password Encoding

Passwords must never be stored in plain text. If an attacker gains access to your database, plain-text passwords expose every user account. Even hashed passwords are vulnerable if the hashing algorithm is weak (MD5, SHA-1) or unsalted.

Spring Security requires a `PasswordEncoder` bean. `BCryptPasswordEncoder` is the standard choice:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

BCrypt is a one-way hashing algorithm designed specifically for passwords. It has two properties that make it suitable for this purpose. It automatically generates a random **salt** for each hash, so the same password produces a different hash every time — this prevents rainbow table attacks. And it is deliberately **slow** — the computational cost is configurable and can be increased as hardware gets faster, making brute-force attacks impractical.

When creating a new user, encode the password before saving it:

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public UserService(UserRepository userRepository,
                       PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    public void registerUser(String username, String rawPassword) {
        AppUser user = new AppUser();
        user.setUsername(username);
        user.setPassword(passwordEncoder.encode(rawPassword));
        user.setRoles(Set.of("USER"));
        userRepository.save(user);
    }
}
```

The `encode` method takes the raw password and produces a BCrypt hash like `$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy`. During authentication, Spring Security calls `passwordEncoder.matches(rawPassword, storedHash)` to compare the login attempt against the stored hash — it never decodes the hash back to the original password.

---

## 8.9 Role-Based Access Control

Authorization determines what authenticated users can do. Spring Security supports authorization at two levels: URL-based rules in the security configuration and method-level annotations.

### URL-Based Authorization

We already saw this in the `SecurityFilterChain`:

```java
http
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/admin/**").hasRole("ADMIN")
        .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
        .requestMatchers("/api/**").hasRole("API_USER")
        .anyRequest().authenticated()
    );
```

Rules are evaluated top to bottom. Put specific rules first and the catch-all `anyRequest()` last.

### Method-Level Authorization with @PreAuthorize

For finer-grained control, enable method-level security and use `@PreAuthorize` on individual service or controller methods:

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
    // ...
}
```

```java
@Service
public class AdminService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }

    @PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
    public UserProfile getProfile(String username) {
        return userRepository.findByUsername(username);
    }
}
```

`@PreAuthorize` evaluates a SpEL (Spring Expression Language) expression before the method executes. If the expression returns `false`, Spring Security throws an `AccessDeniedException` and the method never runs.

The second example shows a powerful pattern: `#username == authentication.name` allows users to access their own profile while restricting access to other users' profiles to administrators. The `#username` refers to the method parameter, and `authentication.name` refers to the currently authenticated user's username.

`@EnableMethodSecurity` activates `@PreAuthorize` support. Without this annotation on a configuration class, `@PreAuthorize` annotations on methods are silently ignored — a common source of confusion.

---

## 8.10 Displaying Security Information in FreeMarker Templates

Unlike Thymeleaf, FreeMarker does not have an official Spring Security dialect. The most practical approach is to expose authentication information through a `@ControllerAdvice` that adds the current user and their roles to the model:

```java
@ControllerAdvice
public class SecurityModelAdvice {

    @ModelAttribute("currentUser")
    public String currentUser() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()
                && !"anonymousUser".equals(auth.getPrincipal())) {
            return auth.getName();
        }
        return null;
    }

    @ModelAttribute("isAdmin")
    public boolean isAdmin() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null) return false;
        return auth.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
    }
}
```

Now every FreeMarker template can conditionally display content based on authentication:

```html
<#-- Show only to authenticated users -->
<#if currentUser??>
    <p>Welcome, ${currentUser}!</p>
</#if>

<#-- Show only to administrators -->
<#if isAdmin>
    <a href="/admin/dashboard">Admin Dashboard</a>
</#if>

<#-- Show only to anonymous users -->
<#if !currentUser??>
    <a href="/login">Login</a>
    <a href="/register">Register</a>
</#if>
```

The `?? ` operator in FreeMarker checks whether a variable exists and is not null. This approach is straightforward, testable, and avoids the complexity of integrating JSP taglibs into FreeMarker.

---

## 8.11 Customizing Login and Logout Pages

### Custom Login Page

Configure the login form in the security filter chain:

```java
.formLogin(form -> form
    .loginPage("/login")
    .loginProcessingUrl("/perform-login")
    .defaultSuccessUrl("/dashboard", true)
    .failureUrl("/login?error=true")
    .permitAll()
)
```

`loginPage("/login")` specifies the URL that displays the login form — you must create a controller mapping and a template for this. `loginProcessingUrl("/perform-login")` is the URL where the form submits credentials — Spring Security handles this automatically, you do not write a controller for it. `defaultSuccessUrl("/dashboard", true)` redirects to `/dashboard` after successful login. `failureUrl("/login?error=true")` redirects back to the login page with an error parameter on failure.

Create a controller that serves the login page:

```java
@Controller
public class AuthController {

    @GetMapping("/login")
    public String loginPage() {
        return "login";
    }
}
```

And the FreeMarker template:

```html
<#import "/spring.ftl" as spring>
<#import "layout.ftlh" as layout>

<@layout.page title="Login">
    <h1>Sign In</h1>

    <#if RequestParameters.error??>
        <p class="error">Invalid username or password.</p>
    </#if>
    <#if RequestParameters.logout??>
        <p class="info">You have been logged out.</p>
    </#if>

    <form action="<@spring.url '/perform-login'/>" method="post">
        <input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}"/>

        <div>
            <label for="username">Username:</label>
            <input type="text" id="username" name="username" required/>
        </div>
        <div>
            <label for="password">Password:</label>
            <input type="password" id="password" name="password" required/>
        </div>
        <button type="submit">Sign In</button>
    </form>
</@layout.page>
```

The CSRF token hidden field is critical — Spring Security rejects any POST request that does not include a valid CSRF token. The `_csrf` object is automatically available in the model when CSRF protection is enabled.

### Custom Logout

```java
.logout(logout -> logout
    .logoutUrl("/logout")
    .logoutSuccessUrl("/login?logout")
    .invalidateHttpSession(true)
    .deleteCookies("JSESSIONID")
    .permitAll()
)
```

The logout form must be a POST request (for CSRF protection):

```html
<form action="<@spring.url '/logout'/>" method="post">
    <input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}"/>
    <button type="submit">Logout</button>
</form>
```

---

## 8.12 User Registration

A registration flow involves a form, validation, password encoding, and saving the user to the database:

```java
@Controller
public class RegistrationController {

    private final UserService userService;

    public RegistrationController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/register")
    public String showRegistrationForm(Model model) {
        model.addAttribute("registrationForm", new RegistrationForm());
        return "register";
    }

    @PostMapping("/register")
    public String registerUser(@Valid @ModelAttribute RegistrationForm form,
                               BindingResult bindingResult,
                               RedirectAttributes redirectAttributes) {
        if (bindingResult.hasErrors()) {
            return "register";
        }

        if (userService.existsByUsername(form.getUsername())) {
            bindingResult.rejectValue("username", "error.username",
                    "Username already exists");
            return "register";
        }

        userService.registerUser(form.getUsername(), form.getPassword());
        redirectAttributes.addFlashAttribute("success",
                "Registration successful! Please log in.");
        return "redirect:/login";
    }
}
```

This follows the same patterns from Chapter 3 — `@Valid` triggers validation, `BindingResult` captures errors, and `RedirectAttributes` carries a flash message through a redirect. The additional check for duplicate usernames happens after validation passes, using `bindingResult.rejectValue` to add a field-level error that the template can display like any other validation error.

---

## 8.13 Securing REST APIs

REST APIs require a different security approach than server-rendered pages. APIs are typically stateless — each request carries its own credentials rather than relying on a server-side session. There is no login page — credentials come in HTTP headers.

You can define a separate `SecurityFilterChain` for API endpoints:

```java
@Bean
public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/products/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .httpBasic(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable());

    return http.build();
}
```

Several things are different from the web page configuration.

`securityMatcher("/api/**")` scopes this filter chain to API endpoints only. You can have multiple filter chains — one for web pages (with form login and CSRF) and another for APIs (with HTTP Basic and no CSRF). Spring Security applies the chain whose `securityMatcher` matches the incoming request.

`httpBasic(Customizer.withDefaults())` enables HTTP Basic authentication, where the client sends credentials in the `Authorization` header with every request. This is simple and appropriate for server-to-server communication or development testing.

`csrf(csrf -> csrf.disable())` disables CSRF protection for the API. CSRF protection defends against browser-based attacks where the browser automatically sends cookies. Stateless APIs that do not use cookies are not vulnerable to CSRF, so the protection is unnecessary. **Note:** In Spring Security 7, CSRF is enabled for all endpoints by default, including APIs. If your API is truly stateless and does not use cookie-based authentication, disabling CSRF for API routes is appropriate.

---

## 8.14 CORS Configuration

**CORS (Cross-Origin Resource Sharing)** controls which domains can call your API from a browser. If your React frontend runs on `http://localhost:3000` and your Spring Boot API runs on `http://localhost:8080`, the browser blocks the API calls by default — they come from different origins. CORS configuration tells the browser which cross-origin requests are allowed.

### Global Configuration

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsConfigurationRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:3000", "https://myfrontend.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

This allows requests from the specified origins to all `/api/**` endpoints using the listed HTTP methods. `allowCredentials(true)` permits the browser to send cookies with cross-origin requests. `maxAge(3600)` tells the browser to cache the CORS preflight response for one hour, reducing the number of preflight OPTIONS requests.

### CORS in the Security Filter Chain

When Spring Security is active, CORS must also be configured in the security filter chain — otherwise the security filter rejects cross-origin preflight requests before they reach the MVC CORS configuration:

```java
http
    .cors(cors -> cors.configurationSource(corsConfigurationSource()))
    // ... other configuration

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

### Per-Controller CORS

For simpler cases, you can use `@CrossOrigin` directly on a controller:

```java
@RestController
@RequestMapping("/api/products")
@CrossOrigin(origins = "http://localhost:3000")
public class ProductRestController {
    // ...
}
```

This is convenient for quick development but does not scale well — global configuration is preferable when multiple controllers need the same CORS policy.

---

## 8.15 Session-Based vs. Token-Based Authentication

Everything we have covered so far uses **session-based authentication** — the server creates a session after login, stores it in memory, and sends a session ID cookie to the browser. The browser sends this cookie with every subsequent request, and the server looks up the session to identify the user.

**Token-based authentication** works differently. After login, the server issues a **JWT (JSON Web Token)** — a self-contained token that includes the user's identity and roles, signed by the server. The client stores the token and sends it in the `Authorization` header with every request. The server validates the token's signature and extracts user information from it — no server-side session storage is needed.

A JWT consists of three Base64-encoded parts separated by dots: a header (algorithm and token type), a payload (claims — user info, roles, expiration time), and a signature (verifies the token has not been tampered with).

### When to Use Each

| Session-Based | Token-Based (JWT) |
|---------------|-------------------|
| Server stores session state. | Server is stateless. |
| Good for traditional server-rendered web applications. | Good for SPAs, mobile apps, and microservices. |
| Uses cookies. | Uses the `Authorization` header. |
| Simpler to implement. | More scalable (no server-side session storage). |
| Logout is straightforward (invalidate session). | Logout requires token blacklisting or short expiration. |

### JWT Flow

The typical JWT authentication flow works as follows: the client sends credentials to a login endpoint (e.g., `POST /api/auth/login`), the server validates the credentials and generates a signed JWT, the client stores the JWT and includes it in the `Authorization` header for subsequent requests (`Authorization: Bearer eyJhbGciOiJIUzI1...`), and the server validates the JWT signature and extracts user info from it on each request.

Full JWT implementation requires additional libraries (such as `jjwt`) and a custom security filter. This is an advanced topic beyond the scope of this chapter, but understanding the concept is important — JWT is the dominant authentication mechanism for modern APIs.

---

## 8.16 Putting It All Together: A Complete Security Configuration

Here is a complete configuration that combines custom user authentication, URL-based authorization, method-level security, form login, and logout:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/home", "/register").permitAll()
                .requestMatchers("/css/**", "/js/**", "/images/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/login?logout")
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
                .permitAll()
            );

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

The `CustomUserDetailsService` from section 8.7 is automatically detected by Spring Boot because it is annotated with `@Service` and implements `UserDetailsService`. No explicit wiring is needed in the security configuration.

The `@EnableMethodSecurity` annotation activates `@PreAuthorize` support so that service methods can enforce their own authorization rules beyond what URL patterns can express.

Static resources (`/css/**`, `/js/**`, `/images/**`) must be explicitly permitted — otherwise Spring Security will require authentication for stylesheets and JavaScript files, which breaks the appearance and functionality of public pages including the login page.

---

## Summary

This chapter covered securing a Spring Boot application with Spring Security 7.

**Common vulnerabilities** — CSRF, XSS, SQL injection, broken authentication — motivate why security cannot be an afterthought. Spring Security provides defenses against many of these out of the box.

**Adding the security starter** immediately secures all endpoints, generates a default user, enables a login form, activates CSRF protection, and sets security headers.

**`SecurityFilterChain`** customizes access rules using `authorizeHttpRequests` with matchers like `permitAll()`, `hasRole()`, `hasAnyRole()`, and `authenticated()`. Rules are evaluated in order — specific rules first, catch-all last.

**Authentication** can use in-memory users (for development), JDBC-backed users (for standard schemas), or a custom `UserDetailsService` (for full control over user loading from your own entity model).

**Password encoding** with `BCryptPasswordEncoder` ensures that passwords are never stored in plain text. BCrypt provides automatic salting and configurable computational cost.

**Role-based access control** enforces authorization at the URL level (in the filter chain) and at the method level (`@PreAuthorize` with `@EnableMethodSecurity`).

**FreeMarker integration** with Spring Security is achieved through a `@ControllerAdvice` that exposes authentication information as model attributes, enabling conditional display of content based on the user's identity and roles.

**Custom login and logout pages** are FreeMarker templates that submit to Spring Security's processing URLs. CSRF tokens must be included as hidden fields in all POST forms.

**REST API security** uses a separate `SecurityFilterChain` scoped with `securityMatcher`, typically with HTTP Basic authentication and CSRF disabled for stateless endpoints.

**CORS configuration** allows cross-origin requests from specified frontend domains, configured both in the MVC layer and in the security filter chain.

**Token-based authentication** with JWT is an alternative to session-based authentication, suited for SPAs, mobile apps, and microservices. It eliminates server-side session storage but requires careful token management.

---

## Resources

- [Spring Security Reference Documentation](https://docs.spring.io/spring-security/reference/index.html)
- [Spring Security Architecture](https://spring.io/guides/topicals/spring-security-architecture/)
- [What's New in Spring Security 7](https://docs.spring.io/spring-security/reference/whats-new.html)
- [OWASP Top Ten](https://owasp.org/www-project-top-ten/)
- [Baeldung: Spring Security](https://www.baeldung.com/security-spring)
- [BCrypt Password Hashing](https://en.wikipedia.org/wiki/Bcrypt)
- [JWT.io](https://jwt.io/)

---

## Lab Assignment: Secure Your Web Application

Add authentication and authorization to your existing web application.

**Requirements:**

1. **Add Spring Security** and configure a `SecurityFilterChain` with URL-based access rules: public pages (home, about, registration) accessible to everyone, protected pages requiring authentication, and admin-only pages accessible only to users with the `ADMIN` role.

2. **Implement user authentication** using a custom `UserDetailsService` backed by a JPA entity and repository. Store passwords encoded with `BCryptPasswordEncoder`.

3. **Create a user registration flow** with a registration form, Bean Validation, duplicate username checking, and password encoding before saving.

4. **Customize the login and logout pages** using FreeMarker templates. Include CSRF tokens in all POST forms. Display error and success messages (invalid credentials, logout confirmation, registration success).

5. **Add conditional content** in your templates based on authentication status — show a welcome message and logout button for authenticated users, show login and register links for anonymous users, and show admin-only navigation for administrators.

6. **Add method-level security** with `@PreAuthorize` on at least one service method — for example, restricting a delete operation to administrators.
