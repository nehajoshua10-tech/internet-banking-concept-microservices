# Internet Banking Config Server

Centralized [Spring Cloud Config](https://spring.io/projects/spring-cloud-config) server for the Internet Banking microservices platform. It exposes configuration that is stored in an external Git repository and serves it, over HTTP, to every other service in the system at startup. This lets the platform keep its configuration externalized, versioned, and managed in one place instead of being duplicated inside each service.

## Key Responsibilities / Features

- Acts as the single source of truth for externalized configuration across the platform.
- Backs its configuration store with a Git repository, so property changes are versioned and auditable.
- Serves configuration to client services through the standard Spring Cloud Config Server HTTP endpoints.
- Exposes Spring Boot Actuator endpoints for health and monitoring.

## Tech Stack

- **Java** (Gradle toolchain, `sourceCompatibility = 1.8`)
- **Spring Boot** 2.6.4
- **Spring Cloud** 2021.0.1 (`spring-cloud-config-server`)
- **Spring Boot Actuator**
- **Gradle** (wrapper included)

Dependencies (from `build.gradle`):

- `org.springframework.cloud:spring-cloud-config-server`
- `org.springframework.boot:spring-boot-starter-actuator`
- `org.springframework.boot:spring-boot-starter-test` (test)

## Configuration

Configuration lives in `src/main/resources/application.yml`.

| Property | Value |
| --- | --- |
| `server.port` | `8090` |
| `spring.cloud.config.server.git.uri` | `https://github.com/javatodev/internet-banking-configurations.git` |
| `spring.cloud.config.server.git.search-paths` | `configuration` |
| `spring.cloud.config.server.git.default-label` | `main` |

Notes:

- The server runs on port **8090**. Every client service points its bootstrap configuration at `http://localhost:8090` to fetch its properties.
- Configuration is loaded from the external Git repository above, from the `configuration` folder on the `main` branch.
- This service does **not** use a database, RabbitMQ, or Keycloak, and it does **not** register itself with the Eureka Service Registry (no Eureka client dependency is present). It is a standalone provider that other services depend on directly by URL.

## Endpoints

This service enables the config server via `@EnableConfigServer` on `InternetBankingConfigServerApplication`. It does not define any custom controllers, Feign clients, or message listeners; it exposes only the standard Spring Cloud Config Server and Actuator endpoints, for example:

- `GET /{application}/{profile}` — configuration for a service and profile
- `GET /{application}/{profile}/{label}` — configuration for a specific Git label
- `GET /{application}-{profile}.yml` / `.properties` — configuration in a specific format
- `GET /actuator/health` — Actuator health endpoint

## Inter-Service Dependencies

- **Upstream:** reads configuration from the external Git repository `javatodev/internet-banking-configurations`.
- **Downstream (consumers):** the config server is a dependency for the other platform services, which pull their configuration from it at boot. These include the API Gateway, User Service, Fund Transfer Service, Utility Payment Service, and the Core Banking Service (each references `http://localhost:8090` in its `bootstrap.yml`).

This service makes no synchronous (OpenFeign) or asynchronous (event) calls to other services itself.

## Build & Run Locally

### Prerequisites

- Java (JDK 8+ compatible; the platform targets Java 11)
- Network access to the external configuration Git repository
- No other service needs to be running for the config server to start; instead, the config server should be started **first** so that the other services can fetch their configuration from it on startup.

### Build

```bash
./gradlew clean build
```

### Run

```bash
./gradlew bootRun
```

Or run the packaged jar:

```bash
./gradlew bootJar
java -jar build/libs/internet-banking-config-server-0.0.1-SNAPSHOT.jar
```

Once running, verify it is up:

```bash
curl http://localhost:8090/actuator/health
```

You can also fetch a service's configuration directly, for example:

```bash
curl http://localhost:8090/internet-banking-api-gateway/default
```
