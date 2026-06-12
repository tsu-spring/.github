# Chapter 14: Deploying Spring Boot Applications

---

## 14.1 From Development to Production

Throughout this course, we have run our application with `mvn spring-boot:run` or from the IDE's run button. We have used an embedded H2 database, watched logs scroll through the console, and tested by clicking through a browser on `localhost:8080`. This is development. Deployment is what happens next — packaging the application, putting it on a server, connecting it to a real database, and making it available to real users.

This final chapter covers the practical skills needed to deploy a Spring Boot application. We will learn how to package the application as an executable JAR, how to write Dockerfiles and build container images, how to use Docker Compose for multi-container setups with a database, how to configure the application for production environments, how to deploy to cloud platforms, and how to set up a basic CI/CD pipeline with GitHub Actions that builds, tests, and deploys automatically on every push.

Spring Boot 4 requires **Java 21** as its effective minimum. All Docker images, build configurations, and CI/CD pipelines in this chapter use Java 21.

---

## 14.2 Packaging: JAR vs. WAR

Spring Boot applications can be packaged as either JAR or WAR files.

| Feature | JAR (Java Archive) | WAR (Web Application Archive) |
|---------|---------------------|-------------------------------|
| Contains | Application + embedded server (Tomcat). | Application only (no server). |
| Run command | `java -jar app.jar` | Deploy to external Tomcat, WildFly, etc. |
| Use case | Modern deployments, Docker, cloud. | Legacy infrastructure, shared servers. |
| Default | Yes. | No (must configure). |

### JAR (Default — Recommended)

Spring Boot creates "fat JARs" (also called "uber JARs") that include all dependencies and an embedded Tomcat server. The JAR is self-contained — it has everything needed to run the application:

```bash
# Build the JAR
mvn clean package

# Run the JAR
java -jar target/myapp-1.0.0.jar
```

This is the deployment model for Docker, cloud platforms, and most modern infrastructure. Use JAR unless you have a specific reason not to.

### WAR (For Legacy Infrastructure)

If you must deploy to an existing Tomcat or WildFly server, package as WAR. This requires three changes:

Change the packaging in `pom.xml`:

```xml
<packaging>war</packaging>
```

Mark Tomcat as `provided` (since the external server supplies it):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

Extend `SpringBootServletInitializer`:

```java
@SpringBootApplication
public class MyApplication extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(MyApplication.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

Then build with `mvn clean package` and copy the resulting `.war` file to Tomcat's `webapps/` directory. The `main` method is retained so the application can still run standalone during development.

---

## 14.3 Building Executable JARs with Maven

### The Maven Build Lifecycle

Maven's build lifecycle is a sequence of phases, each building on the previous:

```bash
mvn clean              # Delete previous build artifacts
mvn compile            # Compile source code
mvn test               # Run tests
mvn package            # Create JAR/WAR (runs compile and test first)
mvn verify             # Run integration tests
mvn install            # Install to local Maven repository
```

The most common command for deployment is `mvn clean package` — it cleans the previous build, compiles the code, runs the tests, and produces the JAR.

### Skipping Tests

When you need a quick build and are confident the tests pass (or plan to run them separately in CI):

```bash
# Skip test execution (tests are still compiled)
mvn package -DskipTests

# Skip test compilation and execution entirely
mvn package -Dmaven.test.skip=true
```

Use these sparingly. Skipping tests in CI defeats the purpose of CI.

### The spring-boot-maven-plugin

This plugin is responsible for creating the executable fat JAR. It is included automatically when you generate a project from Spring Initializr:

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

### Running with Maven

```bash
# Run the application directly (no JAR needed)
mvn spring-boot:run

# Run with a specific profile
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

---

## 14.4 Introduction to Docker

The JAR file contains your application code and dependencies, but it does not contain the Java runtime, the operating system libraries, or the configuration that varies between environments. **Docker** solves this by packaging your application along with its entire runtime environment into a portable **container** that runs identically everywhere — on your laptop, in CI, and in production.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Image** | A read-only blueprint for creating containers (like a class in Java). |
| **Container** | A running instance of an image (like an object). |
| **Dockerfile** | A text file with instructions for building an image. |
| **Docker Hub** | A public registry for sharing images. |
| **Volume** | Persistent storage that survives container restarts. |

### Writing a Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

`FROM eclipse-temurin:21-jre-alpine` starts from an official Java 21 runtime image based on Alpine Linux (a minimal distribution that keeps the image small). We use `jre` rather than `jdk` because the application is already compiled — there is no need for the JDK's compilation tools in the production image.

`COPY target/*.jar app.jar` copies the built JAR into the container.

`EXPOSE 8080` documents which port the application listens on (it does not actually publish the port — that happens at runtime).

`ENTRYPOINT` defines the command that runs when the container starts.

### Building and Running

```bash
# Build the JAR first
mvn clean package -DskipTests

# Build the Docker image
docker build -t myapp:1.0 .

# Run the container
docker run -p 8080:8080 myapp:1.0

# Run in the background
docker run -d -p 8080:8080 --name myapp myapp:1.0

# View logs
docker logs myapp

# Stop and remove
docker stop myapp
docker rm myapp
```

The `-p 8080:8080` flag maps port 8080 on the host to port 8080 inside the container. Without it, the container is running but unreachable.

### Multi-Stage Dockerfile

A multi-stage build compiles the application inside Docker, ensuring a consistent build environment that does not depend on the developer's local setup:

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
COPY mvnw .
COPY .mvn ./.mvn
RUN chmod +x mvnw && ./mvnw clean package -DskipTests

# Stage 2: Run
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

The first stage uses the full JDK to compile and package the application. The second stage uses only the JRE and copies the built JAR from the first stage. The final image does not contain the source code, the Maven cache, or the JDK — only the JRE and the application.

### Production-Ready Dockerfile

For production, add a non-root user (for security) and JVM tuning flags (for container-aware memory management):

```dockerfile
FROM eclipse-temurin:21-jre-alpine

# Run as non-root user
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:MaxRAMPercentage=75.0", \
    "-XX:+UseG1GC", \
    "-jar", "app.jar"]
```

`-XX:MaxRAMPercentage=75.0` tells the JVM to use up to 75% of the container's memory limit, leaving room for the OS and non-heap memory. Without this, the JVM may allocate more memory than the container allows and get killed by the container runtime.

### Passing Environment Variables

Environment variables are how you inject configuration into containers — database URLs, credentials, profiles:

```bash
docker run -p 8080:8080 \
    -e SPRING_PROFILES_ACTIVE=prod \
    -e SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/mydb \
    -e SPRING_DATASOURCE_USERNAME=user \
    -e SPRING_DATASOURCE_PASSWORD=secret \
    myapp:1.0
```

This connects directly to Chapter 5's configuration priority: environment variables override property files, so the same image can run with different databases in different environments.

---

## 14.5 Docker Compose for Multi-Container Setups

A real application runs alongside a database, perhaps a Redis cache, perhaps a monitoring stack. Docker Compose lets you define and run all of these containers together with a single YAML file.

### docker-compose.yml

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/mydb
      - SPRING_DATASOURCE_USERNAME=myuser
      - SPRING_DATASOURCE_PASSWORD=mypassword
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_USER=myuser
      - POSTGRES_PASSWORD=mypassword
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
```

`depends_on` with `condition: service_healthy` tells Docker Compose to wait until PostgreSQL's health check passes before starting the application. Without this, the application might start before the database is ready to accept connections.

The `volumes` section creates a named volume (`postgres-data`) that persists database data across container restarts. Without it, the database is empty every time you restart.

### Docker Compose Commands

```bash
# Start all services (build images if needed)
docker compose up --build

# Start in the background
docker compose up -d

# View logs (follow mode)
docker compose logs -f

# View logs for one service
docker compose logs -f app

# Stop all services
docker compose down

# Stop and remove volumes (deletes database data)
docker compose down -v
```

---

## 14.6 Cloud Deployment

Several cloud platforms offer straightforward deployment for Spring Boot applications. They handle infrastructure, scaling, SSL certificates, and custom domains.

### Railway

[Railway](https://railway.app/) detects Spring Boot projects automatically: connect your GitHub repository, Railway detects the `pom.xml` and builds the project, it sets the `PORT` environment variable automatically, and you can add a PostgreSQL database with one click.

### Render

[Render](https://render.com/) supports Docker-based deployments: create a "Web Service," connect your GitHub repository, Render uses your `Dockerfile` to build and deploy, and you add a managed PostgreSQL database.

### Fly.io

[Fly.io](https://fly.io/) deploys Docker containers globally:

```bash
fly launch      # Initialize the app
fly deploy      # Deploy
fly postgres create    # Add a PostgreSQL database
fly postgres attach    # Attach it to your app
```

### Common Cloud Configuration

Regardless of platform, you typically need to set `SPRING_PROFILES_ACTIVE=prod` as an environment variable, configure database credentials as environment variables, and use a flexible port binding:

```properties
# application-prod.properties
server.port=${PORT:8080}
spring.datasource.url=${DATABASE_URL}
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
```

`${PORT:8080}` reads the port from the `PORT` environment variable (which cloud platforms set automatically), falling back to 8080 if it is not set. `${DATABASE_URL}` reads the database connection string from the environment — cloud platforms inject this when you provision a database.

---

## 14.7 Environment-Specific Configuration in Production

Chapter 5 introduced profiles and environment variables. In production, these are not just convenient — they are essential.

### The Configuration Priority

From lowest to highest priority:

1. Default values in code (`@Value("${key:default}")`)
2. `application.properties` / `application.yml`
3. `application-{profile}.properties` / `application-{profile}.yml`
4. OS environment variables
5. Command-line arguments

Higher-priority sources override lower ones. This means you package sensible defaults in your JAR and override specific values at deploy time:

```bash
# Via command-line argument
java -jar app.jar --spring.profiles.active=prod

# Via environment variable
export SPRING_PROFILES_ACTIVE=prod
java -jar app.jar
```

### A Production Properties File

```properties
# application-prod.properties

# Server
server.port=${PORT:8080}

# Database
spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5432/mydb}
spring.datasource.username=${DB_USER:myuser}
spring.datasource.password=${DB_PASSWORD:mypassword}
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false

# Logging
logging.level.root=WARN
logging.level.com.example=INFO

# Actuator (Chapter 11)
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=when-authorized
```

### Keeping Secrets Safe

Never commit passwords, API keys, or encryption secrets to version control. Instead, use environment variables (as shown throughout this chapter), a secrets manager (AWS Secrets Manager, HashiCorp Vault, Google Secret Manager), or `.env` files for local development (added to `.gitignore`).

```properties
# Structure lives in the property file (committed to Git)
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}

# Values live in the environment (never committed)
```

---

## 14.8 CI/CD with GitHub Actions

Manually building, testing, and deploying every time you push code is tedious and error-prone. **CI/CD** automates this.

**Continuous Integration (CI)** automatically builds and tests the code on every push. If the tests fail, the team knows immediately.

**Continuous Deployment (CD)** automatically deploys to production after the tests pass. The entire path from code change to live deployment is automated.

GitHub Actions uses YAML workflow files stored in `.github/workflows/`.

### A Build and Test Pipeline

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Build and test
        run: mvn clean verify

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: target/surefire-reports/
```

This workflow triggers on every push to `main` or `develop` and on every pull request targeting `main`. It checks out the code, sets up Java 21 (Temurin distribution), caches Maven dependencies to speed up subsequent builds, runs `mvn clean verify` (which compiles, tests, and runs integration tests), and uploads test results as artifacts so you can inspect them if something fails.

### A Build, Test, and Docker Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build and test
        run: mvn clean verify

  docker-build:
    needs: build-and-test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build JAR
        run: mvn clean package -DskipTests

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      # Add deployment steps here:
      # - Push to Docker Hub or GitHub Container Registry
      # - Deploy to Railway, Render, or Fly.io
```

The `needs: build-and-test` directive ensures the Docker build only runs after the tests pass. The Docker image is tagged with the Git commit SHA (`${{ github.sha }}`), which provides a unique, traceable tag for every build.

### Key GitHub Actions Concepts

| Concept | Description |
|---------|-------------|
| `on:` | Events that trigger the workflow (push, pull_request, schedule). |
| `jobs:` | Independent units of work. Run in parallel by default. |
| `steps:` | Sequential actions within a job. |
| `needs:` | Job dependency — run one job after another. |
| `uses:` | Pre-built actions from the GitHub Marketplace. |
| `run:` | Shell commands to execute. |
| `if: always()` | Run the step even if previous steps failed. |

---

## 14.9 A Deployment Checklist

Before deploying to production, walk through this checklist:

**Application configuration.** The production profile is active. Database credentials come from environment variables, not property files. `ddl-auto` is set to `validate` (never `update` or `create` in production). SQL logging is disabled.

**Security.** Actuator endpoints are restricted (Chapter 11). CSRF protection is enabled for web forms. Sensitive error details are hidden (`server.error.include-stacktrace=never`). Passwords are encoded with BCrypt (Chapter 8).

**Monitoring.** The health endpoint is accessible for load balancer probes. Prometheus metrics are exposed for monitoring infrastructure. Structured logging is enabled for log aggregation (Chapter 11).

**Docker.** The container runs as a non-root user. JVM memory is configured relative to the container's memory limit. The image uses a JRE, not a JDK. Health checks are defined.

**CI/CD.** Tests run on every push. The pipeline fails if tests fail. Docker images are tagged with a unique identifier (commit SHA or version number).

---

## Summary

This chapter covered the practical steps for deploying a Spring Boot application to production.

**Packaging** produces either a JAR (self-contained, with embedded Tomcat — the default and recommended choice) or a WAR (for deployment to an external application server). `mvn clean package` builds the artifact, and `java -jar` runs it.

**Docker** packages the application and its runtime environment into a portable container. A Dockerfile defines the image. Multi-stage builds compile inside Docker for consistency. Production images use a JRE base, run as a non-root user, and set JVM memory flags for container-aware behavior. Spring Boot 4 requires **Java 21**, so Docker images use `eclipse-temurin:21-jre-alpine`.

**Docker Compose** defines multi-container setups — application, database, monitoring — in a single YAML file. Health checks ensure services start in the right order. Named volumes persist data across restarts.

**Cloud platforms** (Railway, Render, Fly.io) deploy Spring Boot applications from a GitHub repository or a Dockerfile. They provide managed databases, automatic SSL, and environment variable configuration.

**Production configuration** uses profiles and environment variables to inject database credentials, server ports, and feature flags without modifying code or rebuilding artifacts. Secrets never go in property files — they come from the environment or a secrets manager.

**CI/CD with GitHub Actions** automates building, testing, and deploying on every push. A CI pipeline runs `mvn clean verify` to catch regressions. A CD pipeline builds a Docker image and deploys it after the tests pass.

---

## Resources

- [Spring Boot Docker Guide](https://spring.io/guides/topicals/spring-boot-docker/)
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Railway Documentation](https://docs.railway.app/)
- [Render Documentation](https://render.com/docs)
- [Fly.io Documentation](https://fly.io/docs/)
- [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)

---

## Lab Assignment: Deploy Your Spring Boot Application

Prepare your application for production deployment.

**Requirements:**

1. **Build an executable JAR** using `mvn clean package` and verify it runs with `java -jar`. Confirm that the application starts, serves pages, and connects to a database.

2. **Write a Dockerfile** for your application using `eclipse-temurin:21-jre-alpine` as the base image. Build the Docker image and run it as a container. Verify that the application is accessible at `http://localhost:8080`.

3. **Create a `docker-compose.yml`** that runs your application together with a PostgreSQL database. The application should connect to the PostgreSQL container using environment variables. Use a health check on the database container and `depends_on` with `condition: service_healthy`.

4. **Configure a production profile** (`application-prod.properties`) that reads the database URL, username, and password from environment variables, sets `ddl-auto` to `validate`, and disables SQL logging.

5. **Set up a GitHub Actions workflow** (`.github/workflows/ci.yml`) that builds and tests your application on every push to `main` using Java 21.

6. *(Bonus)* Deploy your application to a cloud platform of your choice (Railway, Render, or Fly.io) and share the live URL.
