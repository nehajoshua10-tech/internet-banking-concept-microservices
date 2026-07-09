# Internet Banking User Service

The **User Service** bridges the core banking system and the Keycloak Identity &
Access Management (IAM) server. It owns the internet-banking user base: it
registers new banking users, validates them against the core banking system,
provisions matching accounts in Keycloak, and drives a **pending-approval
workflow** that only activates a Keycloak account once the user is approved.
Local user records (identification, Keycloak auth id, and approval status) are
persisted in a relational database, while credentials and account state live in
Keycloak.

## Key Responsibilities / Features

- **User registration** – Registers a new banking user by:
  1. Ensuring the email is not already registered in Keycloak.
  2. Looking up the customer in the core banking system by their
     `identification` (via OpenFeign) and verifying the supplied email matches.
  3. Creating a **disabled / email-unverified** user in Keycloak with the given
     password.
  4. Persisting a local record with the Keycloak `authId` and status
     `PENDING`.
- **Pending-approval workflow** – Users are created in a `PENDING` state and
  remain disabled in Keycloak until approved. Updating a user to `APPROVED`
  enables the Keycloak account and marks its email as verified.
- **User retrieval** – Reads a paginated list of users (enriched with email from
  Keycloak) and reads a single user by local id.
- **Keycloak synchronization** – Wraps the Keycloak Admin REST API for creating,
  updating, searching, and reading users within a configured realm.

## Tech Stack

- Java (Spring Boot **2.4.5**, Spring Cloud **2020.0.2**)
- Spring Web (REST controllers)
- Spring Data JPA
- Spring Cloud OpenFeign (declarative REST client)
- Spring Cloud Config Client + Bootstrap
- Spring Cloud Netflix Eureka Client
- Keycloak Admin Client **12.0.4**
- Spring Boot Actuator
- MySQL (runtime JDBC driver); H2 for tests
- Lombok
- Gradle build

## Configuration

This service keeps almost no configuration in its own repository. At startup it
uses a **bootstrap context** to fetch its full configuration from the Spring
Cloud Config Server.

Local files:

- `src/main/resources/application.yml` – only sets the application name:
  ```yaml
  spring:
    application:
      name: internet-banking-user-service
  ```
- `src/main/resources/bootstrap.yml` – points at the Config Server:
  ```yaml
  spring:
    cloud:
      config:
        uri: http://localhost:8090
  ```

Because of this, the **server port, datasource (MySQL), Eureka registry URL, and
Keycloak connection details are all supplied by the Config Server**, which reads
them from the external configuration repository
(`https://github.com/javatodev/internet-banking-configurations`,
`configuration` search path). The Config Server must be running on
`http://localhost:8090` before this service starts.

Keycloak settings are bound from the following properties (see
`KeycloakProperties`), and are expected to be provided by the Config Server:

| Property | Description |
| --- | --- |
| `app.config.keycloak.server-url` | Keycloak server base URL |
| `app.config.keycloak.realm` | Keycloak realm to manage |
| `app.config.keycloak.clientId` | Client id used for admin access |
| `app.config.keycloak.client-secret` | Client secret used for admin access |

The Keycloak Admin client authenticates using the `client_credentials` grant
type.

### Persistence

A JPA entity `UserEntity` is mapped to the `user` table with columns for
`id`, `authId`, `identification`, and `status` (`PENDING` / `APPROVED`, stored
as a string). The MySQL driver is on the runtime classpath; the datasource
itself is provided via the Config Server.

## API Endpoints

All endpoints are served under the base path `/api/v1/bank-user`
(`UserController`). Through the API Gateway they are reachable under the `/user`
prefix (e.g. `/user/api/v1/bank-user`).

| Method | Path | Description | Request body |
| --- | --- | --- | --- |
| `POST` | `/api/v1/bank-user/register` | Register a new banking user (creates the Keycloak account and a local `PENDING` record). | `User` (`email`, `identification`, `password`) |
| `PATCH` | `/api/v1/bank-user/update/{id}` | Update a user's status; setting `APPROVED` enables the Keycloak account. | `UserUpdateRequest` (`status`) |
| `GET` | `/api/v1/bank-user` | Read all users (paginated via `Pageable`), enriched with Keycloak email. | – |
| `GET` | `/api/v1/bank-user/{id}` | Read a single user by local id. | – |

Error responses are standardized (`ErrorResponse`) and use error codes from
`GlobalErrorCode` (e.g. `USER-SERVICE-1001` for an already-registered email,
`USER-SERVICE-1003` when no core-banking user matches the identification).

This service exposes only synchronous REST endpoints; it does **not** publish or
consume any RabbitMQ message channels.

## Inter-Service Dependencies

- **Config Server** (`http://localhost:8090`) – required at startup to load
  configuration.
- **Service Registry (Eureka)** – this service registers itself
  (`@EnableEurekaClient`) so it can be discovered by the API Gateway and resolve
  other services by name.
- **core-banking-service** – called synchronously via OpenFeign
  (`BankingCoreRestClient`, `@FeignClient(name = "core-banking-service")`) at
  `GET /api/v1/user/{identification}` to verify a customer during registration.
  Feign requests use a custom error decoder (`CustomFeignErrorDecoder`).
- **Keycloak** – used through the Keycloak Admin REST API to create, enable,
  search, and read users in the configured realm.

## Build & Run Locally

### Prerequisites

- JDK 11
- A running **Config Server** on `http://localhost:8090`
- A running **Eureka Service Registry**
- A running **Keycloak** server (realm + client configured per the external
  configuration repo)
- A **core-banking-service** instance registered in Eureka
- A **MySQL** database reachable per the Config Server settings

### Commands

From the `internet-banking-user-service/` directory:

```bash
# Build (runs tests)
./gradlew build

# Run the service
./gradlew bootRun
```

Make sure the Config Server and Service Registry are up before starting this
service, as it fetches its configuration and registers itself on startup.
