# Internet Banking API Gateway

The API Gateway is the single, unified entry point for the Internet Banking platform. Built on
Spring Cloud Gateway, it sits in front of the internal microservices and routes incoming HTTP
requests to the appropriate downstream service (user, fund-transfer, core-banking and
utility-payment), abstracting the internal service topology from external clients. It also
enforces authentication/authorization via Keycloak (OAuth2/OIDC) and propagates the
authenticated user's identity to downstream services.

## Key Responsibilities / Features

- **Request routing** — Path-based routing to backend services using client-side load
  balancing (`lb://`) resolved through the Eureka service registry.
- **Security enforcement** — Acts as an OAuth2 Login client and OAuth2 Resource Server
  (JWT) backed by Keycloak. All routes require authentication except the public user
  registration endpoint.
- **Identity propagation** — A custom `GlobalFilter` adds an `X-Auth-Id` header carrying the
  authenticated principal's name (or `SYSTEM USER` when unauthenticated) to every proxied
  request, so downstream services can identify the caller.
- **Path rewriting** — Strips the leading path segment (`StripPrefix=1`) before forwarding, so
  `/user/**` at the gateway maps to `/**` on the user service, etc.
- **Centralized configuration & service discovery** — Reads its configuration from Spring
  Cloud Config Server and registers itself with the Eureka registry.

## Tech Stack

- Java 11 (Gradle `sourceCompatibility = 1.8`)
- Spring Boot 2.4.5
- Spring Cloud 2020.0.4
  - Spring Cloud Gateway (reactive, WebFlux based)
  - Spring Cloud Netflix Eureka Client
  - Spring Cloud Config Client (+ bootstrap)
- Spring Security OAuth2 Client, OAuth2 Resource Server, and OAuth2 JOSE (Keycloak)
- Spring Boot Actuator
- Project Lombok
- Gradle wrapper

## Configuration

The gateway keeps only minimal local configuration; the bulk of its settings are served by the
Config Server.

- **`src/main/resources/application.yml`** sets the application name used for config lookup and
  Eureka registration:
  - `spring.application.name: internet-banking-api-gateway`
- **`src/main/resources/bootstrap.yml`** points at the Config Server so remote configuration is
  fetched during the bootstrap phase:
  - `spring.cloud.config.uri: http://localhost:8090`

The remaining configuration is provided by the Config Server from the external
`internet-banking-configurations` repository (`configuration/internet-banking-api-gateway.yml`)
and includes:

- **Server port:** `8082`
- **Service registry:** Eureka at `http://localhost:8081/eureka` (registers with and fetches
  the registry).
- **Keycloak / OAuth2:** provider, client registration (`internet-banking-core-client`) and
  resource-server JWT settings pointing at a Keycloak realm (default
  `http://localhost:8080/auth`, realm `javatodev-internet-banking`).
- **Gateway routes** (see below).

Because configuration is centralized, the **Config Server (port 8090)** must be running before
the gateway starts, and the **Service Registry (Eureka, port 8081)** plus **Keycloak
(port 8080)** must be reachable for discovery and authentication to work.

This service does **not** use a database or RabbitMQ directly.

## Routes / Endpoints Exposed

The gateway does not expose business REST controllers of its own; it exposes routes that proxy
to downstream services. Routes are defined in the Config Server configuration:

| Path predicate        | Target service (via Eureka)                 | Filter        |
|-----------------------|---------------------------------------------|---------------|
| `/user/**`            | `lb://internet-banking-user-service`        | `StripPrefix=1` |
| `/fund-transfer/**`   | `lb://internet-banking-fund-transfer-service` | `StripPrefix=1` |
| `/banking-core/**`    | `lb://core-banking-service`                 | `StripPrefix=1` |
| `/utility-payment/**` | `lb://internet-banking-utility-payment-service` | `StripPrefix=1` |

Security rules (`SecurityConfiguration`):

- `POST /user/api/v1/register` — permitted without authentication (public registration).
- All other exchanges — authenticated (OAuth2 Login + JWT resource server).
- CSRF is disabled (stateless API gateway).

Actuator endpoints are available (via `spring-boot-starter-actuator`), e.g. downstream health
can be reached through the gateway such as `http://localhost:8082/user/actuator/info` and
`http://localhost:8082/fund-transfer/actuator/info`.

## Inter-Service Dependencies

- **Spring Cloud Config Server** (`:8090`) — required at startup for remote configuration.
- **Eureka Service Registry** (`:8081`) — required for registration and for resolving `lb://`
  route targets.
- **Keycloak** (`:8080`) — OAuth2/OIDC identity provider for login and JWT validation.
- **Downstream microservices** it routes to (resolved by name through Eureka):
  - `internet-banking-user-service`
  - `internet-banking-fund-transfer-service`
  - `core-banking-service`
  - `internet-banking-utility-payment-service`

The gateway communicates with downstream services purely by proxying HTTP requests (via Spring
Cloud Gateway load-balanced routing). It does not use OpenFeign clients, and it neither
publishes nor consumes RabbitMQ events.

## Building and Running Locally

**Prerequisites (start these first):**

1. **Config Server** running on port `8090`.
2. **Eureka Service Registry** running on port `8081`.
3. **Keycloak** running on port `8080` with the `javatodev-internet-banking` realm and the
   `internet-banking-core-client` client configured.
4. Java 11 installed.

**Build:**

```bash
./gradlew clean build
```

**Run:**

```bash
./gradlew bootRun
```

Or run the built jar:

```bash
java -jar build/libs/internet-banking-api-gateway-0.0.1-SNAPSHOT.jar
```

Once started, the gateway listens on **`http://localhost:8082`** and forwards requests to the
downstream services according to the route table above.
