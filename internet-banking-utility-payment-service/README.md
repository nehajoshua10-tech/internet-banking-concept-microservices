# Internet Banking Utility Payment Service

The **Utility Payment Service** is a Spring Boot / Spring Cloud microservice within the
Internet Banking platform. It is responsible for processing utility / biller payments
(for example, paying an electricity, water, or telecom bill) that a customer initiates
from their bank account. It persists each payment request locally, delegates the actual
debit and transaction booking to the `core-banking-service` over REST, and tracks the
lifecycle status of every payment.

## Key Responsibilities / Features

- Accept utility payment requests (provider, amount, reference number, source account).
- Persist each payment attempt with a `TransactionStatus` lifecycle
  (`PENDING` → `PROCESSING` → `SUCCESS` / `FAILED`).
- Forward the payment to the `core-banking-service` to perform the account debit and
  record the transaction, then store the returned `transactionId`.
- Expose a paginated read endpoint to list previously processed utility payments.
- Register with the Eureka service registry and pull its configuration from the Spring
  Cloud Config Server.

## Tech Stack

- **Java** (Gradle `sourceCompatibility = 1.8`)
- **Spring Boot** 2.5.6
- **Spring Cloud** 2020.0.4
  - Netflix Eureka Client (service discovery)
  - OpenFeign (declarative REST client)
  - Spring Cloud Config + Bootstrap (centralized configuration)
- **Spring Data JPA** with **MySQL** (runtime driver `mysql-connector-java`)
- **Spring Boot Actuator**
- **Lombok**
- **H2** (test scope)
- Build tool: **Gradle** (wrapper included)

## Configuration

Local configuration in this repository is intentionally minimal — most runtime settings
are fetched from the Config Server at startup.

`src/main/resources/bootstrap.yml`

```yaml
spring:
  cloud:
    config:
      uri: http://localhost:8090
```

`src/main/resources/application.yml`

```yaml
spring:
  application:
    name: internet-banking-utility-payment-service
```

The remaining settings (server port, datasource, Eureka) are served by the Config Server
from the external configuration repository under the file
`internet-banking-utility-payment-service.yml`:

| Property | Value |
| --- | --- |
| `server.port` | `8085` |
| `spring.datasource.url` | `jdbc:mysql://localhost:3306/banking_core_utility_payment_service` |
| `spring.datasource.username` | `root` |
| `spring.jpa.hibernate.ddl-auto` | `update` |
| `eureka.client.service-url.defaultZone` | `http://localhost:8081/eureka` |

Dependencies at runtime:

- **Spring Cloud Config Server** (`http://localhost:8090`) — required at startup to supply
  the properties above.
- **Eureka Service Registry** (`http://localhost:8081/eureka`) — for registration and for
  resolving the `core-banking-service` used by the Feign client.
- **MySQL** — schema `banking_core_utility_payment_service` for the `utility_payment` table.

> Note: This service does not directly integrate with RabbitMQ or Keycloak; there are no
> messaging or security dependencies declared in its `build.gradle`.

## API Endpoints

Base path: `/api/v1/utility-payment`

| Method | Path | Description | Request body |
| --- | --- | --- | --- |
| `GET` | `/api/v1/utility-payment` | Returns a paginated list of processed utility payments (accepts Spring `Pageable` query params such as `page`, `size`, `sort`). | – |
| `POST` | `/api/v1/utility-payment` | Processes a new utility payment. | `UtilityPaymentRequest` |

`UtilityPaymentRequest`

```json
{
  "providerId": 1,
  "amount": 100.00,
  "referenceNumber": "REF-123456",
  "account": "0000000001"
}
```

`UtilityPaymentResponse`

```json
{
  "message": "Utility Payment Successfully Processed",
  "transactionId": "..."
}
```

This service does not consume or publish any message-broker channels.

## Inter-Service Dependencies

- **`core-banking-service`** — called synchronously via OpenFeign
  (`BankingCoreRestClient`), resolved through Eureka by service name:
  - `POST /api/v1/transaction/util-payment` — performs the debit / books the transaction
    and returns the `transactionId` (used by the payment flow).
  - `GET /api/v1/account/bank-account/{account_number}` — declared account lookup client.
- **Config Server** and **Service Registry** as described in
  [Configuration](#configuration).

## Build & Run Locally

### Prerequisites

Start these first, as this service depends on them at boot time:

1. **Config Server** running on `http://localhost:8090`.
2. **Eureka Service Registry** running on `http://localhost:8081`.
3. **MySQL** running on `localhost:3306` with a database named
   `banking_core_utility_payment_service` (credentials `root` / `password`, or adjust the
   config in the configuration repository).
4. **`core-banking-service`** running and registered with Eureka (required to actually
   process a payment end-to-end).

### Build

```bash
cd internet-banking-utility-payment-service
./gradlew clean build
```

### Run

```bash
./gradlew bootRun
```

The service starts on port **8085** and registers itself with Eureka as
`internet-banking-utility-payment-service`.

### Example request

```bash
curl -X POST http://localhost:8085/api/v1/utility-payment \
  -H "Content-Type: application/json" \
  -d '{
        "providerId": 1,
        "amount": 100.00,
        "referenceNumber": "REF-123456",
        "account": "0000000001"
      }'
```
