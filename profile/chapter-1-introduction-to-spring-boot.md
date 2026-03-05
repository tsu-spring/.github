# Chapter 1: Introduction to Spring & Spring Boot

---

## 1.1 The Spring Ecosystem

Before we write a single line of code, it is worth understanding the landscape we are stepping into. Spring is not a single library or a single framework — it is an entire ecosystem of projects, each designed to solve a particular class of problems that arise when building enterprise Java applications. There are Spring projects for web development, data access, security, messaging, batch processing, cloud infrastructure, and much more. These projects evolve continuously: they receive regular updates, security patches, and new capabilities, and entirely new projects are added to the ecosystem over time.

At the heart of this ecosystem sits the **Spring Framework** itself — a foundational library that provides dependency injection, transaction management, and a programming model for building Java applications. Every other Spring project builds on top of it.

And then there is **Spring Boot**.

---

## 1.2 What Is Spring Boot?

Spring Boot is one of the most widely used projects in the Spring ecosystem. Its purpose is straightforward: to make it as easy as possible to create and run Spring-based applications.

Traditional Spring applications require the developer to write a considerable amount of configuration — XML files, Java configuration classes, servlet mappings, dependency version management, and so on — before any business logic can be written. Spring Boot eliminates most of this ceremony by adopting a philosophy known as **"convention over configuration."** Instead of requiring you to specify every detail, Spring Boot provides sensible default settings. If you need to change something, you override only that specific thing. Everything else stays at its default.

The practical consequence of this philosophy is significant. With Spring Boot, the developer focuses on writing application code rather than wrestling with infrastructure and configuration.

---

## 1.3 Spring Boot vs. Traditional Spring

To appreciate what Spring Boot gives us, it helps to understand what life was like without it.

In a traditional Spring application, you would manually configure a servlet container, define a `DispatcherServlet` in a `web.xml` file (or its Java-based equivalent), declare each bean and its dependencies, manage library versions by hand, and deploy the resulting WAR file to an external application server like Apache Tomcat. Every one of these steps required explicit attention from the developer.

Spring Boot changes this picture in four fundamental ways:

**Auto-Configuration.** When you add a library to your project — say, a web starter or a database driver — Spring Boot detects it on the classpath and automatically configures the necessary beans and settings. You do not need to tell it that you want a web server; it sees the web library and configures one for you.

**Embedded Server.** There is no need to install and manage an external Tomcat instance. Spring Boot embeds a servlet container (Tomcat, by default) directly inside your application. When you run the application, the server starts with it.

**Starter Dependencies.** Instead of hunting down individual libraries and worrying about compatible versions, Spring Boot provides curated "starter" packages. For example, `spring-boot-starter-web` brings in everything needed for web development — Spring MVC, an embedded Tomcat server, JSON processing libraries — in versions that are guaranteed to work together.

**Opinionated Defaults.** Spring Boot chooses defaults that work well out of the box: the server runs on port 8080, templates are loaded from a specific directory, static resources are served from another. These defaults can be overridden at any time, but you only need to do so when your requirements differ from the convention.

---

## 1.4 What Is New in Spring Boot 4

This course uses **Spring Boot 4**, which was released in November 2025. It is built on top of **Spring Framework 7** and represents a new generation of the platform. While the core philosophy of convention over configuration remains unchanged, there are several important updates to be aware of:

**Java 17 Baseline.** Spring Boot 4 requires Java 17 as a minimum. Java 21 or 25 (the latest LTS releases) are recommended to take advantage of newer JVM features such as virtual threads, pattern matching, and records.

**Jakarta EE 11.** The framework has completed its migration from the old `javax.*` packages to `jakarta.*`. If you encounter older tutorials or documentation that reference `javax.servlet` or `javax.persistence`, know that these have been replaced by `jakarta.servlet` and `jakarta.persistence` respectively.

**Modularized Codebase.** The Spring Boot codebase has been split into smaller, more focused modules (jars). This results in a cleaner dependency tree and smaller application footprints.

**Null Safety with JSpecify.** Spring Boot 4 adopts JSpecify annotations across the entire portfolio, helping developers catch potential null-related bugs at compile time.

For this introductory chapter, these changes operate mostly in the background — Spring Boot's auto-configuration handles the details. But it is good to know the terrain, and we will encounter these features in greater depth as the course progresses.

---

## 1.5 Setting Up a Project with Spring Initializr

The fastest way to create a new Spring Boot project is through **Spring Initializr**, a web-based tool provided by the Spring team.

1. Navigate to [https://start.spring.io](https://start.spring.io).
2. Configure the project metadata:
   - **Project:** Maven
   - **Language:** Java
   - **Spring Boot:** 4.0.x (select the latest stable release)
   - **Group:** your organization's reverse domain (e.g., `ge.tsu`)
   - **Artifact:** your project's name (e.g., `blog`)
   - **Packaging:** Jar
   - **Java:** 17 (or 21/25 if available)
3. In the **Dependencies** section, add the following:
   - **Spring Web** — contains the components needed for building web applications (Spring MVC, embedded Tomcat, etc.).
   - **Apache Freemarker** — the FreeMarker template engine, which we will use to render HTML pages.
   - **Spring Boot DevTools** — development-time tools that provide automatic restarts and live reload.
   - **Lombok** — an annotation-processing library that reduces boilerplate code (getters, setters, constructors, etc.).
4. Click **Generate**. A ZIP archive will be downloaded.
5. Extract the archive and open the project in your IDE (IntelliJ IDEA is recommended).
6. Run the `main(..)` method in the generated `*Application.java` class.

If everything is set up correctly, you will see log output ending with a line that says the application has started and is listening on port 8080. Open a browser and navigate to `http://localhost:8080` — you will see a default error page (since we have not defined any content yet), which confirms that the server is running.

---

## 1.6 Understanding the Project Structure

A freshly generated Spring Boot project contains a number of files and directories. Most of them serve the build system or testing infrastructure. For now, the ones that matter most are:

| Path | Purpose |
|------|---------|
| `src/main/java/.../BlogApplication.java` | The main application class. This is the entry point. |
| `src/main/resources/templates/` | FreeMarker template files (`.ftlh`). |
| `src/main/resources/static/` | Static resources — CSS, JavaScript, images. |
| `src/main/resources/application.properties` | The central configuration file. |
| `pom.xml` | Maven project configuration and dependency declarations. |

### The `src/main/resources` Directory

This directory deserves special attention because it is where almost all of our non-Java files will live.

**`/static/`** is intended for files that should be served directly to the browser without any processing: stylesheets, JavaScript files, images, fonts, and so on. When the application starts, Spring Boot makes the contents of this directory available at the root URL. For example, a file placed at `/static/css/style.css` will be accessible at `http://localhost:8080/css/style.css`.

**`/templates/`** is where template files live. In our case, these will be FreeMarker templates with the `.ftlh` extension. Unlike static files, templates are processed by the template engine before being sent to the browser — this is what makes them dynamic.

**`application.properties`** is the main configuration file. Through it, we can override any of Spring Boot's conventional defaults. For example:

```properties
spring.application.name=blog
server.port=8080
server.servlet.context-path=/blog
```

The first line gives our application a name. The second sets the port (8080 is the default, so this line is technically redundant, but being explicit can improve readability). The third sets a context path, meaning the application will be accessible at `http://localhost:8080/blog` instead of the root.

---

## 1.7 Maven and the `pom.xml`

Our project uses **Maven** as its build tool. Maven manages dependencies (the external libraries our project needs), compiles source code, runs tests, and packages the application into an executable JAR file. All of this is configured through a single file: `pom.xml`.

Several parts of the generated `pom.xml` deserve explanation.

### The Parent POM

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.1</version>
    <relativePath/>
</parent>
```

By declaring `spring-boot-starter-parent` as our project's parent, we inherit a great deal of pre-configured behavior: managed dependency versions (so we do not need to specify version numbers for Spring-managed libraries), Maven plugin configuration, the Java version property, and other sensible defaults. This is one of the key mechanisms that makes Spring Boot projects so concise.

If, for some reason, your project already has a different parent POM and cannot use `spring-boot-starter-parent`, you can achieve equivalent dependency management through a BOM (Bill of Materials) import:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>4.0.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Dependencies

The dependencies we selected during project setup appear in the `<dependencies>` section:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

Notice that none of these dependencies specify a version number. The versions are inherited from the parent POM, which ensures that all libraries are compatible with each other.

### The Spring Boot Maven Plugin

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

This plugin is responsible for several important tasks: it can run the application directly from the command line via `mvn spring-boot:run`, it creates executable JAR files (so-called "fat JARs" that bundle all dependencies) via `mvn package`, and it supports the ahead-of-time compilation features introduced in recent Spring Boot versions.

---

## 1.8 The `@SpringBootApplication` Annotation and Auto-Configuration

Every Spring Boot project has a main class that serves as the entry point. It looks like this:

```java
@SpringBootApplication
public class BlogApplication {
    public static void main(String[] args) {
        SpringApplication.run(BlogApplication.class, args);
    }
}
```

The `@SpringBootApplication` annotation is deceptively simple. Behind it, three things are happening:

1. **Component Scanning** (`@ComponentScan`) — Spring scans the package of this class (and all sub-packages) for classes annotated with `@Controller`, `@Service`, `@Repository`, `@Configuration`, and other stereotypes. It registers these classes as beans in the application context.

2. **Auto-Configuration** (`@EnableAutoConfiguration`) — Spring Boot examines the classpath to determine what libraries are present and configures the application accordingly. Because `spring-boot-starter-web` is on the classpath, it automatically sets up an embedded Tomcat server, a `DispatcherServlet`, and default Spring MVC settings. Because `spring-boot-starter-freemarker` is present, it configures the FreeMarker template engine with its default settings (templates in `/templates/`, suffix `.ftlh`, caching enabled in production).

3. **Configuration** (`@Configuration`) — The class itself is a Spring configuration class, meaning it can contain `@Bean` methods if needed.

### How Does Spring Boot Know What to Configure?

Spring Boot ships with a large collection of auto-configuration classes, each responsible for configuring one particular feature or integration. When the application starts, Spring Boot evaluates conditions on each of these classes — for example, "Is this library present on the classpath?" or "Has the developer already defined this bean?" — and activates only the configurations that apply.

This mechanism is what allows Spring Boot to feel "magical" while remaining fully transparent. If you ever need to see exactly what was auto-configured (and what was not), you can add `--debug` to the startup arguments or set `debug=true` in `application.properties`.

### Customizing Auto-Configuration

There are two common ways to customize the auto-configured behavior.

You can **override properties** in `application.properties`:

```properties
server.port=9090
spring.freemarker.suffix=.ftlh
spring.freemarker.cache=false
```

Or you can **exclude specific auto-configurations** entirely:

```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class BlogApplication { ... }
```

The `exclude` attribute is useful when Spring Boot detects a library you have on the classpath but do not actually want auto-configured.

---

## 1.9 Enabling Live Reload with DevTools

The `spring-boot-devtools` dependency we added during project setup exists solely to improve the development experience. Its key features are:

**Automatic Restart.** When you modify a Java class and save the file, DevTools detects the change and restarts the application automatically. This restart is faster than a cold start because DevTools uses two classloaders — one for your code (which is reloaded) and one for third-party libraries (which is not).

**LiveReload.** DevTools includes a LiveReload server. When combined with a LiveReload browser extension, any change to a template, stylesheet, or other resource triggers an automatic browser refresh.

**Development-Friendly Defaults.** Certain properties are adjusted for development. For example, template caching is disabled so that changes to `.ftlh` files are visible immediately without restarting the application.

An important detail: DevTools is automatically disabled when the application is run as a packaged JAR (i.e., in production). The `<scope>runtime</scope>` and `<optional>true</optional>` markers in the `pom.xml` ensure that it does not leak into dependent projects or production builds.

---

## 1.10 Serving Static Content

The simplest kind of website is a static one — every user sees exactly the same content, and no server-side computation is involved. Spring Boot makes it trivial to serve static content.

Any file placed under `src/main/resources/static/` is automatically served at the corresponding URL path. For example:

```
src/main/resources/static/index.html       →  http://localhost:8080/index.html
src/main/resources/static/css/style.css     →  http://localhost:8080/css/style.css
src/main/resources/static/images/logo.png   →  http://localhost:8080/images/logo.png
```

Spring Boot supports four classpath locations for static resources: `/static`, `/public`, `/resources`, and `/META-INF/resources`. In this course we will use `/static` exclusively, for consistency.

A purely static website, however, has a significant limitation: any element that appears on every page — a navigation bar, a header, a footer — must be duplicated in every HTML file. If you want to change the navigation, you must edit every page. This is where a template engine becomes valuable.

---

## 1.11 Introduction to FreeMarker

**Apache FreeMarker** is a template engine for the Java platform. It takes a template file — an HTML document with embedded FreeMarker directives — and combines it with data provided by the application to produce the final HTML that is sent to the browser. FreeMarker templates are written in the **FreeMarker Template Language (FTL)**, a concise language with support for variables, conditionals, loops, macros, and more.

Spring Boot auto-configures FreeMarker when it finds `spring-boot-starter-freemarker` on the classpath. The defaults it applies are:

- Templates are loaded from `src/main/resources/templates/`.
- The expected file extension is `.ftlh` (which enables automatic HTML escaping — a security feature that protects against XSS attacks by escaping special characters in output).
- In production, templates are cached for performance. With DevTools active, caching is disabled.

Because of this auto-configuration, we need to do very little to start using FreeMarker. We place `.ftlh` files in the templates directory, and Spring Boot takes care of the rest.

### FTL Syntax at a Glance

FreeMarker uses two kinds of special syntax in templates:

**Interpolations** — `${expression}` — are used to output values. For example, `${title}` outputs the value of a variable called `title`.

**Directives** — `<#directiveName ...>` — are instructions to the template engine. They control the logic and structure of the template. Common directives include `<#if>`, `<#list>`, `<#assign>`, `<#macro>`, `<#import>`, and `<#include>`.

Comments are written as `<#-- This is a comment -->` and are not included in the output.

---

## 1.12 From Static Site to FreeMarker Templates

Let us walk through the process of converting a static website into one that uses FreeMarker templates, eliminating duplication along the way.

### Step 1: Move HTML Files to `/templates/`

Files that should be processed by FreeMarker must be placed in `src/main/resources/templates/` and given the `.ftlh` extension. Static assets (CSS, JS, images) remain in `/static/`.

```
src/main/resources/
├── static/
│   ├── css/
│   │   └── style.css
│   └── images/
│       └── logo.png
└── templates/
    ├── index.ftlh
    ├── about.ftlh
    ├── posts.ftlh
    └── contact.ftlh
```

### Step 2: Map URLs to Templates

We need to tell Spring Boot which template to render for each URL. For simple pages that do not require any dynamic data from the server, we can use **View Controllers** — a lightweight mechanism that maps a URL directly to a template name without writing a full controller class.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("index");
        registry.addViewController("/about").setViewName("about");
        registry.addViewController("/posts").setViewName("posts");
        registry.addViewController("/contact").setViewName("contact");
    }
}
```

The `@Configuration` annotation tells Spring that this class contains configuration. By implementing `WebMvcConfigurer` and overriding `addViewControllers(..)`, we register URL-to-template mappings. When a user visits `/about`, Spring's `DispatcherServlet` resolves the view name `"about"` to the file `templates/about.ftlh`, passes it through FreeMarker, and returns the result.

Note: the `@Configuration` annotation is essential. Without it, Spring will not recognize this class, and the view controllers will not be registered.

### Step 3: Eliminate Duplication with Macros

At this point, each `.ftlh` file is a complete HTML document. That means the `<head>` section, the navigation bar, and the footer are duplicated across every page. If we change the navigation, we must edit every file.

FreeMarker solves this problem with **macros**. A macro is a reusable template fragment that can accept parameters and render nested content. We will create a **layout macro** — a single macro that defines the entire page structure, with a placeholder where each page's unique content will be inserted.

#### Defining the Layout Macro

Create a file called `layout.ftlh` in the templates directory:

```html
<#macro page title="My Blog">
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>${title}</title>
    <link rel="stylesheet" href="/css/style.css">
</head>
<body>

    <header>
        <nav>
            <a href="/">Home</a>
            <a href="/about">About</a>
            <a href="/posts">Posts</a>
            <a href="/contact">Contact</a>
        </nav>
    </header>

    <main>
        <#nested>
    </main>

    <footer>
        <p>&copy; 2025 My Blog. All rights reserved.</p>
    </footer>

</body>
</html>
</#macro>
```

The key elements here are:

- `<#macro page title="My Blog">` declares a macro named `page` with a parameter `title` that has a default value.
- `<#nested>` is a special directive that renders whatever content the calling template provides between the macro's opening and closing tags.
- `</#macro>` ends the macro definition.

#### Using the Layout Macro

Each page template now becomes dramatically simpler. Here is `index.ftlh`:

```html
<#import "layout.ftlh" as layout>

<@layout.page title="Home">
    <h1>Welcome to My Blog</h1>
    <p>This is the home page of our blog application.</p>
</@layout.page>
```

And `about.ftlh`:

```html
<#import "layout.ftlh" as layout>

<@layout.page title="About Us">
    <h1>About</h1>
    <p>Learn more about who we are and what we do.</p>
</@layout.page>
```

The `<#import "layout.ftlh" as layout>` directive loads the layout file and makes its macros available under the namespace `layout`. The `<@layout.page>` tag invokes the `page` macro, passing a title and providing the page's unique content as the nested body.

This pattern — a layout macro with a `<#nested>` placeholder — is the FreeMarker equivalent of template inheritance in other engines. Every page now has a consistent structure, and any change to the header, footer, or navigation needs to be made in only one place.

---

## 1.13 Spring's Built-In FreeMarker Macros

Spring Framework ships with a set of built-in FreeMarker macros defined in a file called `spring.ftl`. These macros provide convenient integrations between FreeMarker and Spring MVC — most notably for form handling and for generating context-aware URLs.

To use these macros, you import `spring.ftl` at the top of any template (or, more commonly, inside your layout template):

```html
<#import "/spring.ftl" as spring>
```

### The `@spring.url` Macro

One of the most immediately useful macros is `@spring.url`. It generates URLs that are aware of your application's context path. This matters when your application is deployed under a context path other than `/`.

For example, if your `application.properties` contains:

```properties
server.servlet.context-path=/blog
```

Then a plain HTML link like `<a href="/css/style.css">` would break because it does not account for the `/blog` prefix. The `@spring.url` macro solves this:

```html
<link rel="stylesheet" href="<@spring.url '/css/style.css'/>">
<a href="<@spring.url '/'/>">Home</a>
<img src="<@spring.url '/images/logo.png'/>" alt="Logo">
```

If the context path is `/blog`, these will produce `/blog/css/style.css`, `/blog/`, and `/blog/images/logo.png` respectively. If there is no context path, they produce `/css/style.css`, `/`, and `/images/logo.png`. This makes your templates portable regardless of how the application is deployed.

### Integrating `@spring.url` Into the Layout

A practical approach is to import `spring.ftl` inside your layout macro so that it is available to every page without requiring each page to import it separately:

```html
<#import "/spring.ftl" as spring>

<#macro page title="My Blog">
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>${title}</title>
    <link rel="stylesheet" href="<@spring.url '/css/style.css'/>">
</head>
<body>

    <header>
        <nav>
            <a href="<@spring.url '/'/>">Home</a>
            <a href="<@spring.url '/about'/>">About</a>
            <a href="<@spring.url '/posts'/>">Posts</a>
            <a href="<@spring.url '/contact'/>">Contact</a>
        </nav>
    </header>

    <main>
        <#nested>
    </main>

    <footer>
        <p>&copy; 2025 My Blog. All rights reserved.</p>
    </footer>

</body>
</html>
</#macro>
```

We will explore the other Spring macros (particularly the form-related ones such as `@spring.formInput`, `@spring.formSingleSelect`, and `@spring.showErrors`) in later chapters when we begin building forms and handling user input.

---

## 1.14 Conditional Rendering with `<#if>`

Not every part of a page should be rendered every time. Sometimes a section should appear only when certain data is present, or an element should change based on a condition. FreeMarker handles this with the `<#if>` directive.

The basic form is straightforward:

```html
<#if condition>
    ... rendered only when the condition is true ...
</#if>
```

You can add alternative branches with `<#elseif>` and `<#else>`:

```html
<#if user??>
    <p>Welcome back, ${user.name}!</p>
<#else>
    <p>Welcome, guest.</p>
</#if>
```

The `??` operator is a FreeMarker **built-in test** that checks whether a variable exists (is not null). This is one of the most commonly used tests because template data often comes from the server and a value may or may not be present in the model.

Another frequently used test is `?has_content`, which goes a step further: it checks that a value exists **and** is not empty. For strings, this means the string is not null and not `""`. For collections, it means the collection is not null and not empty. This distinction matters — a variable can exist but hold an empty string, and you often want to treat that the same as absent.

```html
<#if subtitle?has_content>
    <h2>${subtitle}</h2>
</#if>
```

Here, the `<h2>` tag is rendered only if `subtitle` is both defined and non-empty. If it is null or `""`, the entire block is skipped.

### Combining `<#if>` with Macros

The real power of conditional rendering becomes apparent inside macros. Consider a blog application where we want a reusable **article preview** component. Some posts have a featured image and some do not. The macro should render the image section only when an image URL is provided:

```html
<#macro article title url imageUrl="">
    <article class="post">
        <#if imageUrl?has_content>
            <section class="post-image">
                <img src="${imageUrl}" alt="post image"/>
            </section>
        </#if>
        <section>
            <h1 class="post-title">
                <a href="${url}">
                    <span>${title}</span>
                </a>
            </h1>
            <main class="post-content">
                <span><#nested></span><a href="${url}">Read more &raquo;</a>
            </main>
        </section>
    </article>
</#macro>
```

There are several things to notice in this macro:

The `imageUrl` parameter has a default value of `""` (an empty string). This means the caller can omit it entirely, and the macro will not break — the `<#if imageUrl?has_content>` test will simply evaluate to false, and the image section will be skipped.

The `<#nested>` directive appears inside the content section, allowing each caller to provide a short excerpt or summary as the macro's body.

Here is how this macro might be used on a posts page:

```html
<#import "components.ftlh" as ui>

<@ui.article title="Getting Started with Spring Boot"
             url="/posts/spring-boot-intro"
             imageUrl="/images/spring-boot.png">
    Spring Boot makes it easy to create stand-alone, production-grade applications...
</@ui.article>

<@ui.article title="Understanding FreeMarker Templates"
             url="/posts/freemarker-templates">
    FreeMarker is a powerful template engine for the Java platform...
</@ui.article>
```

The first article renders with an image. The second one omits the `imageUrl` parameter, so no image section appears — only the title, excerpt, and "Read more" link. Both use the same macro.

---

## 1.15 Reusable Component Macros

The article macro above is one example of a broader pattern: **component macros**. Any reusable HTML pattern — an alert box, a card, a navigation item, a form group — is a candidate for extraction into a macro.

For instance, a simple alert component:

```html
<#macro alert type="info" message="">
    <div class="alert alert-${type}">
        <span>${message}</span>
    </div>
</#macro>
```

Used in a page:

```html
<@alert type="success" message="Your post has been published!" />
<@alert type="error" message="Something went wrong. Please try again." />
```

For better organization, place shared macros in a dedicated file — say, `components.ftlh` — and import them where needed:

```html
<#import "components.ftlh" as ui>

<@ui.alert type="success" message="Saved successfully." />
<@ui.article title="My First Post" url="/posts/first">
    A short preview of the post content...
</@ui.article>
```

This pattern keeps your templates clean and your components reusable. As your application grows, the `components.ftlh` file (or a set of such files, organized by concern) becomes a library of UI building blocks that every page can draw from.

---

## Summary

In this chapter, we covered the foundational concepts needed to begin building web applications with Spring Boot 4 and FreeMarker:

- The **Spring ecosystem** is a collection of projects for enterprise Java development. **Spring Boot** simplifies working with Spring through convention over configuration, auto-configuration, embedded servers, and starter dependencies.
- **Spring Boot 4** requires Java 17+, builds on Spring Framework 7, and targets Jakarta EE 11.
- **Spring Initializr** is the recommended way to generate a new project. Our project uses four starters: `spring-boot-starter-web`, `spring-boot-starter-freemarker`, `spring-boot-devtools`, and `lombok`.
- The **project structure** separates templates (in `/templates/`), static assets (in `/static/`), configuration (in `application.properties`), and build metadata (in `pom.xml`).
- The **`@SpringBootApplication`** annotation triggers component scanning, auto-configuration, and marks the class as a configuration source. Auto-configuration detects classpath libraries and configures beans accordingly.
- **DevTools** provides automatic restarts, LiveReload, and development-friendly defaults. It is automatically disabled in production.
- **Static content** is served from the `/static/` directory without processing.
- **FreeMarker** is our template engine. Templates use the `.ftlh` extension (for HTML auto-escaping), are placed in `/templates/`, and are written in the FreeMarker Template Language (FTL).
- **Macros** — especially the layout macro pattern with `<#nested>` — eliminate HTML duplication by defining a reusable page structure.
- **Spring's built-in macros** (imported via `/spring.ftl`) provide context-aware URL generation with `@spring.url` and form-handling utilities we will use in future chapters.
- **Conditional rendering** with `<#if>`, combined with built-in tests like `??` (exists) and `?has_content` (exists and non-empty), allows templates to adapt their output based on the data they receive.
- **Component macros** — such as the article preview macro — encapsulate reusable UI patterns, keeping templates clean and maintainable as the application grows.

---

## Resources

- [Spring Framework Overview](https://spring.io/projects/spring-framework)
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/index.html)
- [Spring Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes)
- [Spring Initializr](https://start.spring.io)
- [Apache FreeMarker Manual](https://freemarker.apache.org/docs/index.html)
- [FreeMarker Template Language Reference](https://freemarker.apache.org/docs/ref.html)
- [Spring MVC FreeMarker Integration](https://docs.spring.io/spring-framework/reference/web/webmvc-view/mvc-freemarker.html)
- [Spring Web MVC Documentation](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
- [View Controllers Documentation](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/view-controller.html)

---

## Lab Assignment: Build a Static Blog Website

Your task is to create a static blog website using Spring Boot 4 and FreeMarker.

**Requirements:**

1. **Create a new Spring Boot project** using Spring Initializr with the dependencies listed in Section 1.5.

2. **Create at least four pages** — for example: a home page (`index.ftlh`), an about page (`about.ftlh`), a posts page (`posts.ftlh`), and a contact page (`contact.ftlh`). Include whatever content you like; the focus is on structure, not content.

3. **Create a layout macro** in a dedicated file (e.g., `layout.ftlh`) that defines the full HTML document structure — `<!DOCTYPE html>`, `<head>`, navigation, a `<#nested>` placeholder for page content, and a footer. Each page template should use this macro instead of duplicating the document structure.

4. **Use `@spring.url`** for all links and resource references in your templates (CSS files, navigation links, images). Set a custom `server.servlet.context-path` in `application.properties` and verify that all URLs work correctly with the context path.

5. **Create a configuration class** annotated with `@Configuration` that implements `WebMvcConfigurer` and registers view controllers for each of your pages.

6. **Add at least one CSS stylesheet** under `/static/css/` and link it through your layout macro.

7. **Optional: Create a reusable component macro** (e.g., a card, an alert, or a page section) and use it on one or more of your pages.
