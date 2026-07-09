# Core Banking Service

The `core-banking-service` is the **system of record** for the internet-banking platform. It owns the authoritative data for users, bank accounts, utility (biller) accounts, and transactions, and acts as the internal ledger against which balances are held and money movements are recorded.

Higher-level services (fund transfer, utility payment, user service) do not hold balances themselves — they delegate the actual debit/credit and persistence to this service over REST. When a fund transfer or utility payment is executed here, balances are validated and updated and a `TransactionEntity` row is written, making this service the single source of truth for account state.

## Key Responsibilities / Features

- Maintain the core ledger: `banking_core_user`, `banking_core_account`, `banking_core_utility_account`, and `banking_core_transaction` tables.
- Expose read APIs for users, bank accounts, and utility accounts.
- Execute **fund transfers** between two internal bank accounts (balance validation, debit/credit, transaction record).
- Execute **utility payments** from a bank account to a registered utility/biller account.
- Enforce business rules such as insufficient-funds checks, surfaced through standardized error codes (`BANKING-CORE-SERVICE-1000` entity not found, `BANKING-CORE-SERVICE-1001` insufficient funds).
- Manage its own schema via Flyway migrations.

## Tech Stack

- **Java 8** (`sourceCompatibility = 1.8`)
- **Spring Boot 2.5.0** — `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-actuator`
- **Spring Cloud 2020.0.4**
  - `spring-cloud-starter-netflix-eureka-client` (service discovery)
  - `spring-cloud-starter-config` + `spring-cloud-starter-bootstrap` (centralized configuration)
- **Flyway** 8.0.3 for database migrations
- **MySQL** (`mysql-connector-java`) as the runtime datastore
- **Lombok** for boilerplate reduction
- **Gradle** build (`spring-boot` plugin)
- Testing: `spring-boot-starter-test` with **H2** in-memory database

## Configuration

This service uses the **Spring Cloud Config** bootstrap flow. Local files only declare the application name and the config-server location; the rest of the configuration is fetched at startup from the Config Server.

`src/main/resources/application.yml`
```yaml
spring:
  application:
    name: core-banking-service
```

`src/main/resources/bootstrap.yml`
```yaml
spring:
  cloud:
    config:
      uri: http://localhost:8090
```

The remote configuration served for `core-banking-service` (from the Config Server's Git-backed repository) provides:

| Property | Value |
| --- | --- |
| `server.port` | `8092` |
| `spring.datasource.url` | `jdbc:mysql://localhost:3306/banking_core_service` |
| `spring.datasource.username` / `password` | `root` / `password` |
| `spring.jpa.hibernate.ddl-auto` | `none` (schema is managed by Flyway) |
| `eureka.client.service-url.defaultZone` | `http://localhost:8081/eureka` |

Key dependencies at runtime:

- **Config Server** (`http://localhost:8090`) — required at startup to load the properties above.
- **Service Registry / Eureka** (`http://localhost:8081/eureka`) — the app is annotated `@EnableEurekaClient` and registers itself as `core-banking-service`.
- **MySQL** database `banking_core_service` on port `3306`. Flyway applies the migrations in `src/main/resources/db/migration` (base tables, transaction table, and seed/temp data) at startup.

> Note: this service does **not** use RabbitMQ, Keycloak, or OpenFeign directly — it has no messaging, security, or outbound service-call dependencies. Internationalized messages are supported via `messages.properties` / `messages_fr.properties`.

## API Endpoints

All endpoints are served under port `8092` (typically reached through the API Gateway).

### AccountController — `/api/v1/account`

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/api/v1/account/bank-account/{account_number}` | Read a bank account by account number |
| `GET` | `/api/v1/account/util-account/{account_name}` | Read a utility account by provider name |

### UserController — `/api/v1/user`

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/api/v1/user/{identification}` | Read a single user by identification number |
| `GET` | `/api/v1/user` | Read users (paginated via `Pageable`) |

### TransactionController — `/api/v1/transaction`

| Method | Path | Request body | Description |
| --- | --- | --- | --- |
| `POST` | `/api/v1/transaction/fund-transfer` | `FundTransferRequest` (`fromAccount`, `toAccount`, `amount`) | Transfer funds between two internal bank accounts |
| `POST` | `/api/v1/transaction/util-payment` | `UtilityPaymentRequest` (`providerId`, `amount`, `referenceNumber`, `account`) | Pay a utility/biller from a bank account |

The service exposes **no message channels** (no RabbitMQ listeners or publishers).

## Inter-Service Dependencies

- **Consumed by** `internet-banking-fund-transfer-service`, `internet-banking-utility-payment-service`, and `internet-banking-user-service`, which call the REST endpoints above (via OpenFeign / the API Gateway) to read accounts/users and to execute transactions.
- **Depends on** the `internet-banking-config-server` (configuration) and `internet-banking-service-registry` (Eureka discovery) being available at startup.
- This service itself makes **no outbound calls** to other business microservices; all logic operates on its own MySQL datastore.

## Build and Run Locally

### Prerequisites

- JDK 8+
- A MySQL instance reachable at `localhost:3306` with a database named `banking_core_service` (user `root` / password `password`, matching the config-server values or overridden locally).
- The **Config Server** running at `http://localhost:8090`.
- The **Service Registry (Eureka)** running at `http://localhost:8081`.

Start the supporting infrastructure first (`internet-banking-config-server`, then `internet-banking-service-registry`), then start this service.

### Build

```bash
./gradlew clean build
```

### Run

```bash
./gradlew bootRun
```

On startup, Flyway applies the migrations under `src/main/resources/db/migration`, the app registers with Eureka as `core-banking-service`, and it listens on port `8092`.

### Test

```bash
./gradlew test
```

Tests run against an in-memory **H2** database, so no external MySQL/Config Server/Eureka is required to execute them.
