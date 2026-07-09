# Internet Banking Service Registry

The **Internet Banking Service Registry** is the [Netflix Eureka](https://github.com/Netflix/eureka) service discovery server for the Internet Banking microservices platform. It provides a central directory where every other service (API Gateway, Core Banking, User, Fund Transfer, Utility Payment, etc.) registers itself on startup and looks up the network locations of the other services it needs to call. This enables dynamic, location-transparent, client-side service discovery so that services can find one another by logical name instead of hard-coded host/port values.

## Key Responsibilities / Features

- Runs an embedded Eureka server (enabled via `@EnableEurekaServer`).
- Acts as the registry that client services register with and query for discovery.
- Exposes the Eureka dashboard for inspecting registered instances and their health.
- Runs as a standalone (single-node) registry: it does **not** register with, or fetch the registry from, any peer.
- Exposes Spring Boot Actuator endpoints for health and monitoring.

## Tech Stack

- **Java** (Gradle `sourceCompatibility = 1.8`; the provided `Dockerfile` runs the built jar on an `openjdk:11-jre` base image)
- **Spring Boot** 2.4.5
- **Spring Cloud** 2020.0.2 (`spring-cloud-starter-netflix-eureka-server`)
- **Spring Boot Actuator** and **Spring Web**
- **Gradle** 6.8.3 (via the bundled Gradle wrapper)
- **Docker** (`Dockerfile` provided)

## Configuration

Configuration lives in [`src/main/resources/application.yml`](src/main/resources/application.yml):

| Property | Value | Description |
| --- | --- | --- |
| `server.port` | `8081` | Port the registry listens on. |
| `eureka.client.service-url.defaultZone` | `http://localhost:8081/eureka` | Eureka zone URL. |
| `eureka.client.register-with-eureka` | `false` | The registry does not register itself as a client. |
| `eureka.client.fetch-registry` | `false` | The registry does not fetch the registry from a peer. |
| `eureka.instance.prefer-ip-address` | `true` | Prefer IP address over hostname when reporting the instance. |
| `eureka.instance.hostname` | `localhost` | Hostname reported for this instance. |

Notes:

- This service is the discovery server itself, so client services point their `eureka.client.service-url.defaultZone` at `http://localhost:8081/eureka` to register and discover peers.
- It has **no** dependency on the Spring Cloud Config Server — there is no `bootstrap.yml` and no config client dependency; all configuration is local.
- It uses **no** database, RabbitMQ, or Keycloak. It has no domain/business logic.

## API Endpoints / Message Channels

This service defines **no** custom controllers, OpenFeign clients, or message listeners. The endpoints it exposes come from the Eureka server and Spring Boot Actuator:

- **Eureka dashboard:** `http://localhost:8081/` — web UI listing registered instances.
- **Eureka REST API:** under `http://localhost:8081/eureka/*` — used by client services to register, renew, and query instances.
- **Actuator:** `http://localhost:8081/actuator/*` (e.g. `/actuator/health`).

## Inter-Service Dependencies

- **Inbound:** All other services in the platform depend on this registry — they register with it and use it to discover one another. The API Gateway, in particular, relies on it to route requests to backing services by service ID.
- **Outbound:** This service makes **no** synchronous (OpenFeign) calls to other services and publishes/consumes **no** RabbitMQ events. As a standalone registry it has no upstream dependencies and should be started first.

## Build and Run Locally

### Prerequisites

- JDK 11 (recommended; matches the `Dockerfile` runtime).
- No other services are required to start this one — the registry has no upstream dependencies and is typically the **first** service brought up.

### Using the Gradle wrapper

Build:

```bash
./gradlew clean build
```

Run:

```bash
./gradlew bootRun
```

Or run the built jar directly:

```bash
java -jar build/libs/internet-banking-service-registry-0.0.1-SNAPSHOT.jar
```

Once running, open the Eureka dashboard at <http://localhost:8081/>.

### Using Docker

Build the jar first, then build and run the image:

```bash
./gradlew clean build
docker build -t internet-banking-service-registry .
docker run -p 8081:8081 internet-banking-service-registry
```

The container exposes port `8081`.
