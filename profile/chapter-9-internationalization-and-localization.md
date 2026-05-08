# Chapter 9: Internationalization and Localization

---

## 9.1 Making Your Application Multilingual

Every chapter so far has assumed a single language. Our templates display English text, our validation messages are in English, and our error pages speak English. This is fine when your audience shares one language, but many applications need to serve users in different regions — and those users expect the interface in their own language.

This chapter introduces internationalization (i18n) and localization (l10n) in Spring Boot. We will learn how to externalize all user-facing text into message bundles, how to resolve the user's locale, how to let users switch languages, and how to display localized messages in FreeMarker templates. We will also cover localization in REST APIs, in validation messages, in Spring Security pages, and in date and number formatting. Finally, we will look at storing translations in a database for applications that need to manage them dynamically.

---

## 9.2 Internationalization vs. Localization

These two terms are related but distinct.

**Internationalization (i18n)** is the process of designing your application so that it *can* support multiple languages and regions without code changes. This means extracting all user-facing text out of templates and Java code into external files, and using locale-aware formatting for dates, numbers, and currencies. Internationalization is architectural work — you do it once, and it enables localization.

**Localization (l10n)** is the process of actually adapting the application for a specific locale — translating the text, adjusting date formats, and handling regional conventions. Localization is content work — you do it for each language you want to support.

Two things are essential for a localized application: translated text files (one per language) and a mechanism for determining which language the current user wants.

---

## 9.3 Message Bundles

Spring Boot uses **message bundles** — sets of `.properties` files that contain translated text keyed by a common identifier. By default, Spring Boot looks for files named `messages.properties` on the classpath (in `src/main/resources/`).

To support three languages, you create three files:

```
src/main/resources/
├── messages.properties           ← default language (fallback)
├── messages_en.properties        ← English
├── messages_ka.properties        ← Georgian
└── messages_ru.properties        ← Russian
```

Each file contains the same keys with different translated values.

**messages.properties** (default — used as fallback when no locale-specific file matches):

```properties
nav.home=Home
nav.about=About Us
nav.contact=Contact Us
greeting=Welcome to our website!
```

**messages_ka.properties** (Georgian):

```properties
nav.home=\u10DB\u10D7\u10D0\u10D5\u10D0\u10E0\u10D8
nav.about=\u10E9\u10D5\u10D4\u10DC \u10E8\u10D4\u10E1\u10D0\u10EE\u10D4\u10D1
nav.contact=\u10D9\u10DD\u10DC\u10E2\u10D0\u10E5\u10E2\u10D8
greeting=\u10D9\u10D4\u10D7\u10D8\u10DA\u10D8 \u10D8\u10E7\u10DD\u10E1 \u10D7\u10E5\u10D5\u10D4\u10DC\u10D8 \u10DB\u10DD\u10D1\u10E0\u10EB\u10D0\u10DC\u10D4\u10D1\u10D0!
```

**messages_ru.properties** (Russian):

```properties
nav.home=\u0413\u043B\u0430\u0432\u043D\u0430\u044F
nav.about=\u041E \u043D\u0430\u0441
nav.contact=\u041A\u043E\u043D\u0442\u0430\u043A\u0442\u044B
greeting=\u0414\u043E\u0431\u0440\u043E \u043F\u043E\u0436\u0430\u043B\u043E\u0432\u0430\u0442\u044C!
```

The locale suffix follows the ISO 639-1 standard — `en` for English, `ka` for Georgian, `ru` for Russian, `fr` for French, and so on. Spring Boot matches the user's locale to the appropriate file automatically.

### Encoding

The default encoding for `.properties` files is ISO-8859-1, which does not support characters outside the Latin alphabet. For languages like Georgian, Russian, Chinese, or Arabic, you have two options: use Unicode escape sequences (as shown above — `\u10DB` represents a Georgian character), or configure your IDE to save `.properties` files as UTF-8 and add the following to `application.properties`:

```properties
spring.messages.encoding=UTF-8
```

In IntelliJ IDEA, you can configure transparent Unicode-to-ASCII conversion under *Settings → Editor → File Encodings → Properties Files*, which lets you type in the native script while the IDE saves escaped sequences.

---

## 9.4 Displaying Localized Messages in FreeMarker

FreeMarker accesses Spring's message bundles through the `spring.ftl` macros. Import the Spring macro library at the top of your template, then use `<@spring.message>` to look up translated text:

```html
<#import "/spring.ftl" as spring>

<h1><@spring.message "greeting"/></h1>

<nav>
    <a href="/"><@spring.message "nav.home"/></a>
    <a href="/about"><@spring.message "nav.about"/></a>
    <a href="/contact"><@spring.message "nav.contact"/></a>
</nav>
```

When the user's locale is `en`, `<@spring.message "greeting"/>` outputs the value of `greeting` from `messages_en.properties`. When the locale is `ru`, it outputs the Russian translation. If no file matches the locale, the value from the default `messages.properties` is used.

### Messages with Parameters

Message bundles support parameterized messages using `{0}`, `{1}`, etc. as placeholders:

```properties
# messages_en.properties
welcome.user=Welcome, {0}! You have {1} new messages.
```

In the controller, pass the arguments through the model:

```java
@GetMapping("/dashboard")
public String dashboard(Model model, Locale locale) {
    String welcomeMessage = messageSource.getMessage(
            "welcome.user",
            new Object[]{"Alice", 5},
            locale);
    model.addAttribute("welcomeMessage", welcomeMessage);
    return "dashboard";
}
```

```html
<p>${welcomeMessage}</p>
```

The `MessageSource.getMessage` method takes the message key, an array of arguments to substitute into the placeholders, and the locale.

---

## 9.5 Locale Resolution

Spring Boot needs to determine which locale to use for each request. It does this through a **`LocaleResolver`** — a component that reads the locale from somewhere (the request, a session, a cookie) and makes it available to the rest of the application.

### AcceptHeaderLocaleResolver (Default)

By default, Spring Boot uses `AcceptHeaderLocaleResolver`, which reads the locale from the HTTP `Accept-Language` header sent by the browser. This works without any configuration, but the user has no way to override it — they are stuck with whatever language their browser sends.

### SessionLocaleResolver

Stores the locale in the HTTP session. The locale persists for the duration of the session but is lost when the session expires or the server restarts.

### CookieLocaleResolver

Stores the locale in a browser cookie. The locale persists across sessions, server restarts, and even browser restarts — as long as the cookie has not expired. This is usually the best choice for user-facing applications.

| Resolver | Storage | Persistence |
|----------|---------|-------------|
| `AcceptHeaderLocaleResolver` | HTTP header | Cannot be changed by the user. |
| `SessionLocaleResolver` | Server-side session | Lost when the session expires. |
| `CookieLocaleResolver` | Browser cookie | Persists across restarts. |

### Configuring CookieLocaleResolver

```java
@Configuration
public class LocaleConfig {

    @Bean
    public LocaleResolver localeResolver() {
        CookieLocaleResolver resolver = new CookieLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        resolver.setCookieMaxAge(Duration.ofDays(30));
        resolver.setCookiePath("/");
        return resolver;
    }
}
```

`setDefaultLocale(Locale.ENGLISH)` specifies the language used when no cookie exists yet. `setCookieMaxAge(Duration.ofDays(30))` keeps the cookie alive for 30 days. `setCookiePath("/")` makes the cookie available across the entire application.

---

## 9.6 Switching Languages with LocaleChangeInterceptor

A `LocaleResolver` stores the locale, but something needs to trigger a change. The `LocaleChangeInterceptor` watches for a request parameter (typically `lang`) and updates the locale when it is present.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");
        return interceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
    }
}
```

With this configuration, appending `?lang=ru` to any URL switches the locale to Russian. The `CookieLocaleResolver` saves this choice in a cookie, so the user does not have to switch again on subsequent pages.

---

## 9.7 Building a Language Switcher in the UI

The user needs a visible way to change languages. A dropdown that redirects to the current page with the `?lang=` parameter is the simplest approach:

```html
<select id="change-language">
    <option value="" selected><@spring.message "language.switch"/></option>
    <option value="ka">ქართული</option>
    <option value="en">English</option>
    <option value="ru">Русский</option>
</select>

<script>
    document.querySelector("#change-language").addEventListener("change", function(e) {
        if (e.target.value) {
            window.location.href = window.location.pathname + '?lang=' + e.target.value;
        }
    });
</script>
```

When the user selects a language, JavaScript redirects to the current page with the `?lang=` parameter. The `LocaleChangeInterceptor` picks up the parameter, the `CookieLocaleResolver` stores it, and the page re-renders in the new language.

You can style this as a dropdown, as flag icons, as text links — whatever fits your design. The mechanism is the same: navigate to any URL with `?lang=XX`.

---

## 9.8 Localizing Validation Messages

Validation messages from Bean Validation annotations (Chapter 3) can be localized by referencing message bundle keys instead of hardcoded strings. Use curly braces `{...}` in the annotation's `message` attribute:

```java
@NotBlank(message = "{user.name.notblank}")
private String name;

@Email(message = "{user.email.invalid}")
private String email;

@Size(min = 6, message = "{user.password.size}")
private String password;
```

Then provide translations in each message bundle:

**messages_en.properties:**

```properties
user.name.notblank=Name is required.
user.email.invalid=Please provide a valid email address.
user.password.size=Password must be at least 6 characters.
```

**messages_ka.properties:**

```properties
user.name.notblank=\u10E1\u10D0\u10EE\u10D4\u10DA\u10D8 \u10E1\u10D0\u10D5\u10D0\u10DA\u10D4\u10D1\u10D8\u10E1\u10DD\u10D0.
user.email.invalid=\u10D2\u10D7\u10EE\u10DD\u10D5\u10D7 \u10DB\u10D8\u10E3\u10D7\u10D8\u10D7\u10D4\u10D7 \u10E1\u10EC\u10DD\u10E0\u10D8 \u10D4\u10DA\u10D4\u10E5\u10E2\u10E0\u10DD\u10DC\u10E3\u10DA\u10D8 \u10E4\u10DD\u10E1\u10E2\u10D8\u10E1 \u10DB\u10D8\u10E1\u10D0\u10DB\u10D0\u10E0\u10D7\u10D8.
user.password.size=\u10DE\u10D0\u10E0\u10DD\u10DA\u10D8 \u10E3\u10DC\u10D3\u10D0 \u10E8\u10D4\u10D8\u10EA\u10D0\u10D5\u10D3\u10D4\u10E1 \u10DB\u10D8\u10DC\u10D8\u10DB\u10E3\u10DB 6 \u10E1\u10D8\u10DB\u10D1\u10DD\u10DA\u10DD\u10E1.
```

The curly braces `{user.name.notblank}` in the annotation tell the validation framework to look up the key in the message bundle for the current locale. Without the curly braces, the string is used literally.

For this to work, Spring Boot's `MessageSource` must be used by the validator. Spring Boot auto-configures this by default — you do not need any additional setup.

---

## 9.9 Localization in REST APIs

REST APIs do not render templates, so they cannot use FreeMarker's `<@spring.message>` macro. Instead, inject `MessageSource` directly and resolve messages programmatically:

```java
@RestController
@RequestMapping("/api")
public class GreetingController {

    private final MessageSource messageSource;

    public GreetingController(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    @GetMapping("/greet")
    public String greet(Locale locale) {
        return messageSource.getMessage("greeting", null, locale);
    }
}
```

The `Locale` parameter is resolved automatically by Spring from the request — either from the `Accept-Language` header or from the `LocaleResolver`. You can test this with curl:

```bash
curl -H "Accept-Language: ru" http://localhost:8080/api/greet
```

This returns the Russian translation of the `greeting` message.

---

## 9.10 Localization with Spring Security

The login and error pages from Chapter 8 can be localized using the same message bundle mechanism. Define the security-related text in your message bundles:

**messages_en.properties:**

```properties
security.login.title=Sign In
security.login.username=Username
security.login.password=Password
security.login.submit=Sign In
security.login.error=Invalid username or password.
security.logout.success=You have been logged out.
```

**messages_ka.properties:**

```properties
security.login.title=\u10E8\u10D4\u10E1\u10D5\u10DA\u10D0
security.login.username=\u10DB\u10DD\u10DB\u10EE\u10DB\u10D0\u10E0\u10D4\u10D1\u10DA\u10D8\u10E1 \u10E1\u10D0\u10EE\u10D4\u10DA\u10D8
security.login.password=\u10DE\u10D0\u10E0\u10DD\u10DA\u10D8
security.login.submit=\u10E8\u10D4\u10E1\u10D5\u10DA\u10D0
security.login.error=\u10D0\u10E0\u10D0\u10E1\u10EC\u10DD\u10E0\u10D8 \u10DB\u10DD\u10DB\u10EE\u10DB\u10D0\u10E0\u10D4\u10D1\u10D4\u10DA\u10D8 \u10D0\u10DC \u10DE\u10D0\u10E0\u10DD\u10DA\u10D8.
security.logout.success=\u10D7\u10E5\u10D5\u10D4\u10DC \u10D2\u10D0\u10DB\u10DD\u10EE\u10D5\u10D4\u10D3\u10D8\u10D7.
```

Then use these keys in the login template:

```html
<#import "/spring.ftl" as spring>

<h1><@spring.message "security.login.title"/></h1>

<#if RequestParameters.error??>
    <p class="error"><@spring.message "security.login.error"/></p>
</#if>
<#if RequestParameters.logout??>
    <p class="info"><@spring.message "security.logout.success"/></p>
</#if>

<form action="<@spring.url '/perform-login'/>" method="post">
    <input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}"/>

    <label for="username"><@spring.message "security.login.username"/>:</label>
    <input type="text" id="username" name="username" required/>

    <label for="password"><@spring.message "security.login.password"/>:</label>
    <input type="password" id="password" name="password" required/>

    <button type="submit"><@spring.message "security.login.submit"/></button>
</form>
```

There is nothing special about localizing security pages — they use the same `<@spring.message>` macro as every other template.

---

## 9.11 Formatting Dates, Numbers, and Currencies

Localization goes beyond translating text. Dates, numbers, and currencies are formatted differently across regions: the United States writes `$1,234.56` and `June 15, 2025`, while Germany writes `1.234,56 €` and `15. Juni 2025`.

### Formatting in Java

Use `java.text.NumberFormat` and `java.time.format.DateTimeFormatter` with a locale:

```java
import java.text.NumberFormat;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.time.format.FormatStyle;
import java.util.Locale;

// Currency formatting
NumberFormat currencyFormat = NumberFormat.getCurrencyInstance(locale);
String price = currencyFormat.format(19.99);
// "19,99 €" (de), "$19.99" (en-US), "19,99 ₽" (ru)

// Date formatting
DateTimeFormatter dateFormat = DateTimeFormatter
        .ofLocalizedDate(FormatStyle.LONG)
        .withLocale(locale);
String date = LocalDate.now().format(dateFormat);
// "15. Juni 2025" (de), "June 15, 2025" (en-US), "15 июня 2025 г." (ru)
```

Pass the formatted strings through the model to the template. This gives you full control over the formatting and keeps locale-sensitive logic in Java where it is testable.

### Formatting in FreeMarker

FreeMarker provides built-in formatting directives that respect the locale set on the template configuration:

```html
<#-- Number formatting -->
<p>${price?string(",##0.00")}</p>

<#-- Date formatting -->
<p>${createdAt?string("dd MMMM yyyy")}</p>

<#-- Currency (using locale-aware Java formatting from the model) -->
<p>${formattedPrice}</p>
```

For locale-aware currency and date formatting, the most reliable approach is to format values in Java (in a service or controller) and pass the formatted strings to the template. FreeMarker's built-in formatting respects the template locale, but Java's `NumberFormat` and `DateTimeFormatter` give you more control and consistency.

---

## 9.12 Storing Translations in a Database

For most applications, file-based message bundles are sufficient. But some applications need to manage translations dynamically — adding new languages, updating translations without redeploying, or letting non-developers edit translations through an admin interface. In these cases, you can store messages in a database and implement a custom `MessageSource`.

### The Database Schema

```sql
CREATE TABLE messages (
    id SERIAL PRIMARY KEY,
    code VARCHAR(255) NOT NULL,
    locale VARCHAR(10) NOT NULL,
    message TEXT NOT NULL
);
```

### A Custom MessageSource

```java
public class DatabaseMessageSource extends AbstractMessageSource {

    private final JdbcTemplate jdbcTemplate;

    public DatabaseMessageSource(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    protected MessageFormat resolveCode(String code, Locale locale) {
        String message;
        try {
            message = jdbcTemplate.queryForObject(
                    "SELECT message FROM messages WHERE code = ? AND locale = ?",
                    String.class,
                    code, locale.toString());
        } catch (EmptyResultDataAccessException e) {
            message = null;
        }
        return (message != null) ? new MessageFormat(message, locale) : null;
    }
}
```

### Registering the Custom MessageSource

```java
@Configuration
public class MessageSourceConfig {

    private final JdbcTemplate jdbcTemplate;

    public MessageSourceConfig(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Bean
    public MessageSource messageSource() {
        DatabaseMessageSource source = new DatabaseMessageSource(jdbcTemplate);
        source.setUseCodeAsDefaultMessage(true);
        return source;
    }
}
```

`setUseCodeAsDefaultMessage(true)` tells the message source to return the key itself (e.g., `nav.home`) as the message if no translation is found, rather than throwing an exception. This is useful as a fallback during development while translations are incomplete.

This approach queries the database for every message lookup. In a production application, you would add caching — either using Spring's `@Cacheable` or an in-memory cache that is refreshed periodically — to avoid a database query for every message on every page render.

---

## 9.13 Putting It All Together

Here is how the configuration classes, message bundles, and templates connect:

### LocaleConfig.java

```java
@Configuration
public class LocaleConfig {

    @Bean
    public LocaleResolver localeResolver() {
        CookieLocaleResolver resolver = new CookieLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        resolver.setCookieMaxAge(Duration.ofDays(30));
        resolver.setCookiePath("/");
        return resolver;
    }
}
```

### WebConfig.java

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");
        return interceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
    }
}
```

### A Localized Template

```html
<#import "/spring.ftl" as spring>
<#import "layout.ftlh" as layout>

<@layout.page title="Home">
    <h1><@spring.message "greeting"/></h1>

    <nav>
        <a href="/"><@spring.message "nav.home"/></a>
        <a href="/about"><@spring.message "nav.about"/></a>
        <a href="/contact"><@spring.message "nav.contact"/></a>
    </nav>

    <select id="change-language">
        <option value="" selected><@spring.message "language.switch"/></option>
        <option value="ka">ქართული</option>
        <option value="en">English</option>
        <option value="ru">Русский</option>
    </select>

    <script>
        document.querySelector("#change-language").addEventListener("change", function(e) {
            if (e.target.value) {
                window.location.href = window.location.pathname + '?lang=' + e.target.value;
            }
        });
    </script>
</@layout.page>
```

The flow is: the user selects a language from the dropdown, JavaScript redirects with `?lang=ru`, the `LocaleChangeInterceptor` picks up the parameter, the `CookieLocaleResolver` stores it in a cookie, and the page re-renders with all `<@spring.message>` calls resolving against `messages_ru.properties`.

---

## Summary

This chapter covered making a Spring Boot application multilingual through internationalization and localization.

**Message bundles** (`messages.properties`, `messages_en.properties`, `messages_ru.properties`) externalize all user-facing text. Each file contains the same keys with translations for a specific locale. Spring Boot loads the appropriate file based on the current locale.

**FreeMarker integration** uses the `<@spring.message "key"/>` macro (from `spring.ftl`) to display localized text in templates. Parameterized messages use `{0}`, `{1}` placeholders resolved through `MessageSource.getMessage`.

**Locale resolution** determines which locale to use for each request. `CookieLocaleResolver` stores the locale in a browser cookie for persistence across sessions. `SessionLocaleResolver` stores it in the HTTP session. `AcceptHeaderLocaleResolver` reads it from the browser's `Accept-Language` header.

**`LocaleChangeInterceptor`** watches for a `?lang=` request parameter and updates the locale through the `LocaleResolver`. A language switcher in the UI triggers this by redirecting to the current page with the parameter.

**Validation messages** can be localized by referencing message bundle keys with `{curly.braces}` in Bean Validation annotations.

**REST API localization** injects `MessageSource` directly and resolves messages programmatically using the `Locale` parameter.

**Date, number, and currency formatting** varies by region. Java's `NumberFormat` and `DateTimeFormatter` with a locale produce correctly formatted strings for each region.

**Database-backed translations** use a custom `MessageSource` implementation that queries a `messages` table, enabling dynamic translation management without redeployment.

---

## Resources

- [Internationalization — Spring Boot](https://docs.spring.io/spring-boot/reference/features/internationalization.html)
- [FreeMarker Spring Macros — Spring Framework](https://docs.spring.io/spring-framework/reference/web/webmvc-view/mvc-freemarker.html)
- [Guide to Internationalization in Spring Boot — Baeldung](https://www.baeldung.com/spring-boot-internationalization)
- [Custom Validation MessageSource in Spring Boot — Baeldung](https://www.baeldung.com/spring-custom-validation-message-source)
- [ISO 639-1 Language Codes — Wikipedia](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes)

---

## Lab Assignment: Build a Multilingual Application

Apply internationalization and localization to your web application.

**Requirements:**

1. **Set a default language** — Georgian or your native language — and add at least one additional language (English, Russian, or any language of your choice).

2. **Create message bundle files** (`messages.properties`, `messages_en.properties`, etc.) in `src/main/resources/`. Externalize all user-facing text: navigation labels, page titles, form labels, button text, and any other text that users see.

3. **Localize validation messages** by referencing message bundle keys in your Bean Validation annotations (`@NotBlank(message = "{key}")`). Verify that validation errors display in the selected language.

4. **Implement a language switcher** in the UI — a dropdown, flag icons, or buttons — that lets users change the locale at any time by navigating with a `?lang=` parameter.

5. **Configure locale persistence** using `CookieLocaleResolver` so the selected language persists across pages and browser sessions.

6. **Register a `LocaleChangeInterceptor`** in a `WebMvcConfigurer` so the `?lang=` parameter is intercepted and applied to the `LocaleResolver`.

7. *(Optional)* Localize your Spring Security login and logout pages, and format dates or numbers in a locale-aware manner.
