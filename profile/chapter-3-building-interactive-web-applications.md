# Chapter 3: Building Interactive Web Applications

---

## 3.1 From Display to Interaction

In the first two chapters, we built pages that display information — static content, template-rendered layouts, and data passed from controllers through the model. The user could view content, but could not send anything back. A real web application, however, is a conversation: the user submits data, the server processes it, and the application responds accordingly.

This chapter introduces the mechanisms that make that conversation possible. We will learn how to handle form submissions, bind form data to Java objects, validate user input on the server side, display validation errors back to the user, handle file uploads, and prevent the duplicate-submission problem with the Post/Redirect/Get pattern. We will also create custom error pages.

All form rendering in this chapter uses **Spring's built-in FreeMarker macros** — the `spring.ftl` library we first encountered in Chapter 1 for URL generation. These macros provide form binding, input generation, and error display that integrates directly with Spring MVC's data binding and validation infrastructure.

---

## 3.2 Handling HTTP GET and POST Requests

Every interaction with a web application begins with an HTTP request. Two request methods dominate form-based web applications:

**GET** requests retrieve data. When you navigate to a URL, click a link, or type an address in the browser bar, a GET request is sent. GET requests should never modify data — they are safe and idempotent.

**POST** requests submit data. When you fill in a form and click "Submit," a POST request is sent with the form data in the request body. POST requests typically create or modify a resource on the server.

In Spring MVC, handler methods in a `@Controller` class are mapped to specific HTTP methods using `@GetMapping` and `@PostMapping`:

```java
@Controller
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping("/products")
    public String listProducts(Model model) {
        model.addAttribute("products", productService.findAll());
        return "products/list";
    }

    @PostMapping("/products")
    public String createProduct(@ModelAttribute Product product) {
        productService.save(product);
        return "redirect:/products";
    }
}
```

The same URL — `/products` — is handled by two different methods. Spring dispatches to the correct one based on the HTTP method. The GET handler retrieves data and renders a template. The POST handler processes the submitted form data and redirects (we will discuss why it redirects rather than rendering directly in Section 3.8).

---

## 3.3 Building Forms with Spring's FreeMarker Macros

To build an HTML form that integrates with Spring MVC's data binding, we use the macros defined in `spring.ftl`. These macros generate form input elements that are automatically bound to a form-backing object (also called a "command object") in the model.

### Preparing the Form in the Controller

Before the form can be rendered, the GET handler must add an empty object to the model. This object is what Spring will bind the form fields to:

```java
@GetMapping("/products/new")
public String showCreateForm(Model model) {
    model.addAttribute("product", new Product());
    return "products/create";
}
```

The key `"product"` is the name by which the form-backing object is known. In the template, all form macro paths will start with this name.

### The Form Template

Here is a complete form using Spring's FreeMarker macros:

```html
<#import "/spring.ftl" as spring>
<#import "layout.ftlh" as layout>

<@layout.page title="Create Product">
    <h1>Create a New Product</h1>

    <form action="<@spring.url '/products'/>" method="post">
        <div>
            <label for="name">Name:</label>
            <@spring.formInput "product.name" 'id="name" class="form-control"'/>
            <@spring.showErrors "<br>" "error"/>
        </div>

        <div>
            <label for="price">Price:</label>
            <@spring.formInput "product.price" 'id="price" class="form-control"' "number"/>
            <@spring.showErrors "<br>" "error"/>
        </div>

        <div>
            <label for="description">Description:</label>
            <@spring.formTextarea "product.description" 'id="description" rows="5" class="form-control"'/>
            <@spring.showErrors "<br>" "error"/>
        </div>

        <button type="submit">Create Product</button>
    </form>
</@layout.page>
```

Let us break down what is happening.

`<@spring.formInput "product.name" 'id="name" class="form-control"'/>` generates an `<input type="text">` element. The first argument — `"product.name"` — is the **path**: the name of the form-backing object (`product`) followed by the field name (`name`). The macro binds to this path, meaning it will populate the input's `value` attribute from the object's `name` property, and name the input so that submitted data maps back to that property. The second argument is a string of additional HTML attributes to include in the generated tag.

`<@spring.formInput "product.price" '...' "number"/>` demonstrates the optional third argument — the field type. By passing `"number"`, the macro generates `<input type="number">` instead of the default text input. You can also pass `"hidden"` or `"password"` here.

`<@spring.formTextarea "product.description" '...'/>` generates a `<textarea>` element bound to the `description` field.

`<@spring.showErrors "<br>" "error"/>` displays validation errors for the most recently bound field. The first argument is a separator between multiple error messages. The second argument is a CSS class name applied to the `<span>` wrapping each error. If omitted, errors are wrapped in `<b>` tags instead.

### Available Form Macros

Spring's `spring.ftl` provides the following macros for form generation:

| Macro | Purpose |
|-------|---------|
| `<@spring.formInput path attrs fieldType/>` | Text input (default), or number/hidden/password via `fieldType`. |
| `<@spring.formTextarea path attrs/>` | Multi-line text area. |
| `<@spring.formPasswordInput path attrs/>` | Password input (never pre-filled). |
| `<@spring.formHiddenInput path attrs/>` | Hidden input. |
| `<@spring.formSingleSelect path options attrs/>` | Drop-down select (single value). |
| `<@spring.formMultiSelect path options attrs/>` | Multi-select list box. |
| `<@spring.formRadioButtons path options separator attrs/>` | Radio button group. |
| `<@spring.formCheckboxes path options separator attrs/>` | Checkbox group. |
| `<@spring.formCheckbox path attrs/>` | Single checkbox. |
| `<@spring.showErrors separator classOrStyle/>` | Display validation errors for the last bound field. |
| `<@spring.message code/>` | Output a message from a resource bundle. |
| `<@spring.url relativeUrl/>` | Context-aware URL (covered in Chapter 1). |

For selection macros (`formSingleSelect`, `formRadioButtons`, etc.), the `options` parameter is a `Map` where keys are the values submitted with the form and values are the labels displayed to the user. This map is typically provided by the controller:

```java
@ModelAttribute("categories")
public Map<String, String> populateCategories() {
    Map<String, String> categories = new LinkedHashMap<>();
    categories.put("electronics", "Electronics");
    categories.put("books", "Books");
    categories.put("clothing", "Clothing");
    return categories;
}
```

And used in the template:

```html
<div>
    <label>Category:</label>
    <@spring.formSingleSelect "product.category" categories 'class="form-control"'/>
    <@spring.showErrors "<br>" "error"/>
</div>
```

---

## 3.4 Data Binding with `@ModelAttribute`

When the user submits the form, the browser sends a POST request with the form field values encoded in the request body. Spring MVC automatically maps these values to a Java object through a process called **data binding**.

The `@ModelAttribute` annotation on a method parameter tells Spring to create an instance of the specified class, populate its fields from the request parameters (matching by name), and pass it to the handler method:

```java
@PostMapping("/products")
public String createProduct(@ModelAttribute Product product) {
    // 'product' is automatically populated:
    //   product.name = value from the "name" input
    //   product.price = value from the "price" input
    //   product.description = value from the "description" textarea
    productService.save(product);
    return "redirect:/products";
}
```

The binding works by matching form field names (which the Spring macros set correctly based on the path) to the object's properties. Spring handles type conversion automatically — it converts the string `"29.99"` from the form into a `Double` for the `price` field, for example.

`@ModelAttribute` can also be used at the method level to add attributes to the model for every request handled by the controller. We saw this above with the `populateCategories` method — it runs before every handler method in the controller and adds the categories map to the model.

---

## 3.5 Server-Side Validation with Bean Validation

Accepting whatever the user sends without checking it is a recipe for bad data, crashes, and security vulnerabilities. Server-side validation ensures that data meets our requirements before it is processed.

Spring Boot supports **Bean Validation** (the Jakarta Validation specification, formerly JSR 380) through the Hibernate Validator implementation. To use it, add the validation starter to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### Annotating the Model Class

Validation rules are declared directly on the fields of your model class using annotations:

```java
public class Product {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be positive")
    private Double price;

    @Size(max = 500, message = "Description cannot exceed 500 characters")
    private String description;

    // getters and setters
}
```

The most commonly used validation annotations are:

| Annotation | Rule |
|------------|------|
| `@NotNull` | Must not be null. |
| `@NotBlank` | Must not be null, empty, or whitespace (strings only). |
| `@NotEmpty` | Must not be null or empty (strings, collections). |
| `@Size(min, max)` | String length or collection size must be within range. |
| `@Min(value)` / `@Max(value)` | Numeric value must meet the bound. |
| `@Positive` / `@Negative` | Must be a positive / negative number. |
| `@Email` | Must be a valid email format. |
| `@Pattern(regexp)` | Must match the regular expression. |
| `@Past` / `@Future` | Date must be in the past / future. |

### Triggering Validation in the Controller

To activate validation, add `@Valid` before the `@ModelAttribute` parameter and include a `BindingResult` parameter immediately after it:

```java
@PostMapping("/products")
public String createProduct(@Valid @ModelAttribute Product product,
                            BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        return "products/create";  // Re-display the form with errors
    }
    productService.save(product);
    return "redirect:/products";
}
```

When Spring encounters `@Valid`, it runs all the validation annotations on the object before the handler method executes. If any constraint is violated, the errors are collected in the `BindingResult`. The handler method then checks `bindingResult.hasErrors()` — if there are errors, it returns the form template name (without a redirect), which causes the form to be re-rendered. Because the form-backing object is still in the model (with the user's submitted values and the errors), the Spring macros will repopulate the fields and the `<@spring.showErrors>` macro will display the error messages.

**Important:** `BindingResult` must be declared immediately after the `@Valid` parameter in the method signature. If any other parameter appears between them, Spring will not bind the errors correctly.

---

## 3.6 Displaying Validation Errors

The `<@spring.showErrors>` macro, which we already used in Section 3.3, is the primary way to display validation errors in FreeMarker templates. It displays errors for the field that was most recently bound — that is, the field used in the most recent `<@spring.formInput>`, `<@spring.formTextarea>`, or similar macro call.

Here is a complete form with validation error display:

```html
<#import "/spring.ftl" as spring>
<#import "layout.ftlh" as layout>

<@layout.page title="Create Product">
    <h1>Create a New Product</h1>

    <form action="<@spring.url '/products'/>" method="post">
        <div class="form-group">
            <label for="name">Name:</label>
            <@spring.formInput "product.name" 'id="name" class="form-control"'/>
            <@spring.showErrors "<br>" "text-danger"/>
        </div>

        <div class="form-group">
            <label for="price">Price:</label>
            <@spring.formInput "product.price" 'id="price" class="form-control"' "number"/>
            <@spring.showErrors "<br>" "text-danger"/>
        </div>

        <div class="form-group">
            <label for="description">Description:</label>
            <@spring.formTextarea "product.description" 'id="description" rows="5" class="form-control"'/>
            <@spring.showErrors "<br>" "text-danger"/>
        </div>

        <button type="submit" class="btn">Create Product</button>
    </form>
</@layout.page>
```

When validation fails, the `<@spring.showErrors "<br>" "text-danger"/>` calls generate HTML like:

```html
<span class="text-danger">Name is required</span>
```

The error messages come from the `message` attribute of the validation annotations on the model class. The CSS class (`"text-danger"` in this example) allows you to style error messages to stand out visually — typically in red.

### Displaying All Errors at Once

If you prefer to show a summary of all errors at the top of the form rather than next to each field, you can use the `<@spring.bind>` macro directly and iterate over the error messages:

```html
<@spring.bind "product"/>
<#if spring.status.errors?has_content>
    <div class="alert alert-danger">
        <ul>
            <#list spring.status.errors.allErrors as error>
                <li>${error.defaultMessage}</li>
            </#list>
        </ul>
    </div>
</#if>
```

The `<@spring.bind "product"/>` macro binds to the form-backing object as a whole. The `spring.status` object then gives access to all errors across all fields.

---

## 3.7 Handling File Uploads

Many applications need to accept file uploads — profile pictures, document attachments, cover images. Spring Boot makes this straightforward with `MultipartFile`.

### Configuration

Set upload size limits in `application.properties`:

```properties
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

### The Upload Form

File upload forms must use `enctype="multipart/form-data"`. Since file inputs are not bound to a form-backing object in the same way as text fields, we use plain HTML for the file input:

```html
<form action="<@spring.url '/upload'/>" method="post" enctype="multipart/form-data">
    <div>
        <label for="file">Choose file:</label>
        <input type="file" id="file" name="file"/>
    </div>
    <button type="submit">Upload</button>
</form>
```

### The Controller

The uploaded file arrives as a `MultipartFile` parameter, resolved via `@RequestParam`:

```java
@Controller
public class FileUploadController {

    @PostMapping("/upload")
    public String handleFileUpload(@RequestParam("file") MultipartFile file,
                                   RedirectAttributes redirectAttributes) {
        if (file.isEmpty()) {
            redirectAttributes.addFlashAttribute("message", "Please select a file.");
            return "redirect:/upload";
        }

        try {
            byte[] bytes = file.getBytes();
            Path path = Paths.get("uploads/" + file.getOriginalFilename());
            Files.createDirectories(path.getParent());
            Files.write(path, bytes);

            redirectAttributes.addFlashAttribute("message",
                    "Uploaded successfully: " + file.getOriginalFilename());
        } catch (IOException e) {
            redirectAttributes.addFlashAttribute("message",
                    "Upload failed: " + e.getMessage());
        }

        return "redirect:/upload";
    }
}
```

`MultipartFile` provides methods for accessing the file's content (`getBytes()`, `getInputStream()`), its original filename (`getOriginalFilename()`), its content type (`getContentType()`), and its size (`getSize()`). Always validate the file before saving — check that it is not empty, that its content type is acceptable, and that its size is within your limits.

---

## 3.8 The Post/Redirect/Get Pattern

Consider what happens when a user submits a form. The browser sends a POST request. The server processes it. If the server responds with an HTML page directly (by returning a template name), the browser's address bar still shows the POST URL, and the browser's last request was a POST. If the user presses the refresh button, the browser will resend the POST — potentially creating a duplicate entry, charging a credit card twice, or sending a message again.

The **Post/Redirect/Get (PRG)** pattern prevents this. Instead of responding to the POST with HTML directly, the server responds with an HTTP 302 redirect to a GET URL. The browser follows the redirect and issues a GET request. Now, if the user refreshes, only the safe GET request is repeated.

The flow is:

1. User submits the form → **POST** `/products`
2. Server processes the data, then responds with → **302 Redirect** to `/products`
3. Browser follows the redirect → **GET** `/products`
4. User refreshes → only the GET is repeated (safe)

In Spring MVC, a redirect is triggered by returning a string that starts with `"redirect:"`:

```java
@PostMapping("/products")
public String createProduct(@Valid @ModelAttribute Product product,
                            BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        return "products/create";  // No redirect — re-display form with errors
    }
    productService.save(product);
    return "redirect:/products";   // PRG: redirect to GET endpoint
}
```

Notice that when validation fails, we do *not* redirect — we return the template name directly. This is deliberate. The validation errors and the form-backing object (with the user's submitted data) are in the current request's model. A redirect would lose them. Only on success do we redirect.

---

## 3.9 Flash Attributes

The PRG pattern creates a problem: a redirect is a new HTTP request, and model attributes from the original POST request are lost. But sometimes you need to pass a message to the redirected page — for example, "Product created successfully!" This is what **flash attributes** solve.

Flash attributes are stored temporarily in the HTTP session. They survive exactly one redirect and are then removed automatically. Spring provides them through the `RedirectAttributes` parameter:

```java
@PostMapping("/products")
public String createProduct(@Valid @ModelAttribute Product product,
                            BindingResult bindingResult,
                            RedirectAttributes redirectAttributes) {
    if (bindingResult.hasErrors()) {
        return "products/create";
    }
    productService.save(product);
    redirectAttributes.addFlashAttribute("successMessage", "Product created successfully!");
    return "redirect:/products";
}
```

In the target template, the flash attribute is available as a regular model variable:

```html
<#if successMessage??>
    <div class="alert alert-success">
        ${successMessage}
    </div>
</#if>
```

After the page is rendered, the flash attribute is automatically removed from the session. If the user refreshes the page, the message will not appear again — which is exactly the right behavior.

---

## 3.10 Custom Error Pages

When something goes wrong — a page is not found, a server error occurs — Spring Boot displays a default error page. In a production application, you will want to replace this with custom pages that match your application's design.

### Convention-Based Error Pages

The simplest approach is to place FreeMarker templates in `/templates/error/`, named after HTTP status codes:

```
src/main/resources/templates/error/
├── 404.ftlh       ← displayed for 404 Not Found
├── 500.ftlh       ← displayed for 500 Internal Server Error
└── error.ftlh     ← generic fallback for any other error
```

Spring Boot automatically maps HTTP error status codes to these templates. No additional configuration is needed.

A `404.ftlh` template might look like this:

```html
<#import "/spring.ftl" as spring>
<#import "../layout.ftlh" as layout>

<@layout.page title="Page Not Found">
    <h1>404 — Page Not Found</h1>
    <p>Sorry, the page you are looking for does not exist.</p>
    <p><a href="<@spring.url '/'/>">Return to the home page</a></p>
</@layout.page>
```

Spring Boot makes several attributes available in error templates: `timestamp`, `status`, `error` (the reason phrase), `message`, and `path` (the URL that triggered the error). You can use these to provide more detail:

```html
<p>Status: ${status!""}</p>
<p>Error: ${error!""}</p>
<p>Timestamp: ${timestamp!""}</p>
```

The `!""` syntax is FreeMarker's default value operator — if the variable is missing, it outputs an empty string instead of throwing an error.

### Custom ErrorController

For more control over error handling logic — for example, to add model attributes, log the error, or apply different logic based on the status code — you can implement the `ErrorController` interface:

```java
@Controller
public class CustomErrorController implements ErrorController {

    @RequestMapping("/error")
    public String handleError(HttpServletRequest request, Model model) {
        Object status = request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);

        if (status != null) {
            int statusCode = Integer.parseInt(status.toString());
            model.addAttribute("statusCode", statusCode);

            if (statusCode == HttpStatus.NOT_FOUND.value()) {
                return "error/404";
            } else if (statusCode == HttpStatus.INTERNAL_SERVER_ERROR.value()) {
                return "error/500";
            }
        }
        return "error/error";
    }
}
```

This controller intercepts all error responses. It reads the HTTP status code from the request, adds it to the model, and routes to the appropriate error template. You can extend this with logging, custom messages, or any other logic you need.

The convention-based approach and the custom `ErrorController` can coexist — the `ErrorController` takes precedence when it is present, but falls back to the convention-based templates for any status codes it does not explicitly handle.

### Exception Handling with @ControllerAdvice

The convention-based pages and `ErrorController` handle generic HTTP errors — 404 from missing routes, 500 from unexpected failures. But as your application grows, you will have **domain-specific** exceptions: a requested item does not exist, a user is not authorized, a business rule is violated. For these, Spring provides a more targeted mechanism: `@ControllerAdvice` with `@ExceptionHandler`.

A `@ControllerAdvice` class acts as a global interceptor for exceptions thrown by any controller. Each method annotated with `@ExceptionHandler` specifies which exception type it catches:

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public String handleNotFound(ProductNotFoundException ex, Model model) {
        model.addAttribute("message", ex.getMessage());
        return "error/404";
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public String handleGenericError(Exception ex, Model model) {
        model.addAttribute("message", "Something went wrong. Please try again later.");
        return "error/500";
    }
}
```

When a `ProductNotFoundException` is thrown anywhere in the application — in a controller, in a service called by a controller — Spring intercepts it before the response is sent, sets the HTTP status to 404, adds the exception's message to the model, and renders the `error/404` template. The catch-all `Exception` handler ensures that any unexpected error produces a user-friendly 500 page instead of a raw stack trace.

The exception classes themselves are simple:

```java
public class ProductNotFoundException extends RuntimeException {

    public ProductNotFoundException(Long id) {
        super("Product with ID " + id + " was not found.");
    }
}
```

This approach is more powerful than `ErrorController` for several reasons. It gives you **type-safe** exception handling — different exceptions can produce different responses with different status codes and different templates. It also keeps error-handling logic **close to the domain** — you can see exactly which error scenarios your application handles and how. And because `@ControllerAdvice` participates in Spring's full MVC pipeline, your exception handler methods have access to the `Model`, can return view names, and work seamlessly with your templates.

In practice, `@ControllerAdvice` is the most common approach in Spring applications. You will typically use it alongside convention-based error pages: the `@ControllerAdvice` handles domain exceptions you anticipate, and the convention-based pages serve as a fallback for anything else.

---

## Summary

This chapter covered the mechanisms that turn a display-only website into an interactive application.

**HTTP GET and POST** requests serve different purposes: GET retrieves data, POST submits it. Spring MVC's `@GetMapping` and `@PostMapping` map handler methods to the appropriate request type.

**Spring's FreeMarker form macros** (`@spring.formInput`, `@spring.formTextarea`, `@spring.formSingleSelect`, `@spring.showErrors`, and others) generate HTML form elements that are bound to a form-backing object in the model. They handle populating fields with existing values and displaying validation errors automatically.

**Data binding** with `@ModelAttribute` maps submitted form data to Java object properties by name. Spring handles type conversion automatically.

**Bean Validation** annotations (`@NotBlank`, `@Size`, `@Positive`, `@Email`, etc.) declare validation rules directly on model classes. The `@Valid` annotation triggers validation, and `BindingResult` captures errors. When errors exist, the form is re-displayed with error messages rendered by `<@spring.showErrors>`.

**File uploads** use `MultipartFile` and require `enctype="multipart/form-data"` on the form. Configuration properties control maximum file and request sizes.

**The Post/Redirect/Get pattern** prevents duplicate form submissions by redirecting after a successful POST. **Flash attributes** preserve messages across the redirect by temporarily storing them in the session.

**Custom error pages** can be created by placing templates in `/templates/error/` named after status codes, or by implementing `ErrorController` for programmatic control. **`@ControllerAdvice`** with `@ExceptionHandler` provides the most flexible approach — it catches domain-specific exceptions globally and routes them to the appropriate error template with the correct HTTP status code.

---

## Resources

- [Spring MVC — Annotated Controllers](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller.html)
- [Spring MVC — FreeMarker Form Macros](https://docs.spring.io/spring-framework/reference/web/webmvc-view/mvc-freemarker.html)
- [Bean Validation (Hibernate Validator)](https://hibernate.org/validator/)
- [Spring Boot — File Upload Guide](https://spring.io/guides/gs/uploading-files/)
- [Spring Boot — Error Handling](https://docs.spring.io/spring-boot/reference/web/servlet.html#web.servlet.spring-mvc.error-handling)
- [Apache FreeMarker Manual](https://freemarker.apache.org/docs/index.html)

---

## Lab Assignment: Build an Interactive Web Application

Extend your blog application with interactive features.

**Requirements:**

1. **Create an HTML form** for adding a new blog post (title, author, content, and optionally a category selected from a dropdown). Use Spring's FreeMarker form macros (`@spring.formInput`, `@spring.formTextarea`, `@spring.formSingleSelect`) to bind the form to a model object.

2. **Implement server-side validation** using Bean Validation annotations. The title should be required and between 3 and 100 characters. The content should be required. Display validation errors next to each field using `<@spring.showErrors>`.

3. **Use the Post/Redirect/Get pattern** — on successful submission, redirect to the posts list page. On validation failure, re-display the form with errors and the user's previously entered data preserved.

4. **Add flash attributes** to display a success message (e.g., "Post created successfully!") on the posts list page after a successful redirect.

5. **Implement file upload** — allow users to attach an image to a blog post. Validate that the uploaded file is not empty and is an image type.

6. **Create custom error pages** — at minimum, create a `404.ftlh` and a generic `error.ftlh` page that use your layout macro for consistent styling.
