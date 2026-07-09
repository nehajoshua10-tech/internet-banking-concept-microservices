# Internet Banking Fund Transfer Service

The **Fund Transfer Service** orchestrates inter-account money movement (transfers between bank accounts) within the internet-banking microservices platform. It accepts fund-transfer requests over REST, persists a local record of every transfer with its status, and delegates the actual debit/credit posting to the `core-banking-service` (the system of record) via an OpenFeign client. Local records let the service track the lifecycle of each transfer (`PENDING` → `SUCCESS`) and expose transfer history, while the core banking service owns the authoritative ledger.

## Key Responsibilities / Features

- Accept fund-transfer requests (from account, to account, amount, authorization ID).
- Persist each transfer locally as a `fund_transfer` record with a `TransactionStatus` (`PENDING`, `SUCCESS`, `FAILED`).
- Post the transfer to `core-banking-service` via OpenFeign and capture the returned transaction reference.
- Expose transfer history with pagination.
- Register with the Eureka service registry for discovery and pull its configuration from the Spring Cloud Config Server.

## Tech Stack

- **Java** (Spring Boot `2.4.5`, Spring Cloud `2020.0.2`)
- **Spring Boot Web** — REST controllers
- **Spring Data JPA** + **MySQL** (`mysql-connector-java`) — local persistence
- **Spring Cloud OpenFeign** — declarative REST client to `core-banking-service`
- **Spring Cloud Config** (`spring-cloud-starter-config` / `spring-cloud-starter-bootstrap`) — externalized configuration
- **Netflix Eureka Client** — service discovery
- **Spring Boot Actuator** — health/info endpoints
- **Lombok** — boilerplate reduction
- **H2** — in-memory database for tests
- **Gradle** — build tool

## Configuration

Configuration is split between the packaged resources and the centralized Config Server.

Bundled in the service (`src/main/resources`):

- `spring.application.name = internet-banking-fund-transfer-service`
- `spring.cloud.config.uri = http://localhost:8090` (Config Server) — set in `bootstrap.yml`

Served remotely by the Config Server (`internet-banking-fund-transfer-service.yml` in the configurations repo):

| Property | Value |
| --- | --- |
| `server.port` | `8084` |
| `spring.datasource.url` | `jdbc:mysql://localhost:3306/banking_core_fund_transfer_service` |
| `spring.datasource.username` | `root` |
| `spring.datasource.password` | `password` |
| `spring.jpa.hibernate.ddl-auto` | `update` |
| `eureka.client.service-url.defaultZone` | `http://localhost:8081/eureka` |

Notes:

- **Config Server** (port `8090`) must be reachable at startup — the bootstrap context fetches remote config before the app context starts.
- **Service Registry** (Eureka, port `8081`) is used for registration and to resolve `core-banking-service`.
- **MySQL** database `banking_core_fund_transfer_service` must exist (schema is auto-managed via `ddl-auto: update`).
- This service does **not** integrate with RabbitMQ or Keycloak directly.

## API Endpoints

Base path: `/api/v1/transfer`

| Method | Path | Description | Request Body |
| --- | --- | --- | --- |
| `POST` | `/api/v1/transfer` | Initiate a fund transfer. Persists a `PENDING` record, posts to core banking, then marks it `SUCCESS`. Returns a `FundTransferResponse`. | `FundTransferRequest` |
| `GET` | `/api/v1/transfer` | Read fund transfers (paginated via Spring `Pageable`, e.g. `?page=0&size=20`). Returns a list of `FundTransfer`. | — |

**`FundTransferRequest`**

```json
{
  "fromAccount": "string",
  "toAccount": "string",
  "amount": 0.00,
  "authID": "string"
}
```

**`FundTransferResponse`**

```json
{
  "message": "Fund Transfer Successfully Completed",
  "transactionId": "string"
}
```

Errors are handled by a global `@ControllerAdvice`: `SimpleBankingGlobalException` returns HTTP 400 with an `ErrorResponse` (`code`, `message`); other exceptions return HTTP 400 with an error string.

## Inter-Service Dependencies

- **`core-banking-service`** (synchronous, via OpenFeign — `BankingCoreFeignClient`):
  - `POST /api/v1/transaction/fund-transfer` — posts the transfer and returns the transaction reference.
  - `GET /api/v1/account/bank-account/{account_number}` — reads a bank account.
  - The Feign client is resolved by service name through Eureka and uses `CustomFeignClientConfiguration` (full request/response logging).
- **`internet-banking-config-server`** — supplies runtime configuration at bootstrap.
- **`internet-banking-service-registry`** (Eureka) — service registration and discovery.

This service does not publish or consume any RabbitMQ / messaging events.

## Build & Run Locally

### Prerequisites

- JDK 11+
- MySQL running on `localhost:3306` with a `banking_core_fund_transfer_service` database
- **Config Server** running on `http://localhost:8090`
- **Service Registry (Eureka)** running on `http://localhost:8081/eureka`
- **core-banking-service** running and registered (required for transfers to succeed)

### Commands

From the `internet-banking-fund-transfer-service/` directory:

```bash
# Build
./gradlew clean build

# Run
./gradlew bootRun
```

Or run the packaged jar:

```bash
./gradlew bootJar
java -jar build/libs/internet-banking-fund-transfer-service-0.0.1-SNAPSHOT.jar
```

The service starts on port **8084**. Ensure the Config Server and Service Registry are up first, otherwise startup will fail while resolving remote configuration and registering with Eureka.
