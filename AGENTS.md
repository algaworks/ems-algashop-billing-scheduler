# AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the **Billing Scheduler Service**, a lightweight Spring Boot application responsible for scheduled invoice expiration and cancellation. It runs as a standalone service that periodically queries for expired unpaid invoices and cancels them via the Fastpay payment gateway.

**Key Responsibilities:**
- Query for expired unpaid invoices (older than 1 day)
- Cancel expired invoices via Fastpay payment gateway
- Update invoice status to CANCELED with audit trail
- Batch processing with SKIP LOCKED for concurrent safety

**Technology Stack:**
- Java 25, Gradle 9.2.1, Spring Boot 4.0.x
- PostgreSQL (with Flyway migrations)
- Spring JDBC (raw SQL, no JPA)
- Spring REST Client (declarative HTTP client)
- Fastpay payment gateway integration

## Architecture

### Scheduled Task Pattern

Simple, focused architecture for a single scheduled operation:

```
application/      → Application service interface (business logic contract)
infrastructure/   → JDBC implementation, Fastpay HTTP client, configuration
```

**Layers:**

1. **Application Layer** (`application/`)
    - `CancelExpiredInvoicesApplicationService` — Interface defining the cancellation contract

2. **Infrastructure Layer** (`infrastructure/`)
    - `CancelExpiredInvoicesApplicationServiceJDBCImpl` — Queries DB via JDBC, orchestrates cancellation
    - `CancelExpiredInvoicesRunner` — `ApplicationRunner` triggered on startup; logs execution
    - `FastpayPaymentAPIClient` — Declarative HTTP client (`@PutExchange`) for Fastpay cancel endpoint
    - `FastpayFastpayPaymentAPIClientConfig` — Builds `RestClient` with auth interceptor (Token header)
    - `AlgaShopPaymentProperties` — `@ConfigurationProperties` for Fastpay hostname and private token
    - `InvoiceProjection` — DTO: `invoiceId`, `paymentCode`

## Build Commands

```bash
cd microservices/billing-scheduler

# Compile and run all tests
./gradlew build

# Run unit tests only (*Test.java)
./gradlew test

# Run integration tests (*IT.java)
./gradlew integrationTest

# Run all test types
./gradlew check

# Build runnable JAR
./gradlew bootJar

# Build multi-platform Docker image
./gradlew dockerBuild
```

## Running Locally

**Start infrastructure:**
```bash
cd ../..
docker compose -f docker-compose.tools.yml up -d
```

**Run the application:**
```bash
./gradlew bootRun
```

On startup, `CancelExpiredInvoicesRunner` executes immediately and logs results.

**Fastpay config (development):**
- Fastpay hostname: `http://localhost:9995`
- Private token: configured in `application-development-env.yml`

## Project Structure

```
src/main/java/com/algaworks/algashop/billingscheduler/
├── BillingSchedulerApplication.java
├── application/
│   └── CancelExpiredInvoicesApplicationService.java
└── infrastructure/
    ├── AlgaShopPaymentProperties.java
    ├── CancelExpiredInvoicesApplicationServiceJDBCImpl.java
    ├── CancelExpiredInvoicesRunner.java
    ├── FastpayFastpayPaymentAPIClientConfig.java
    ├── FastpayPaymentAPIClient.java
    └── InvoiceProjection.java
```

## Database

Uses same `billing` database (shared with billing service).

**Invoices table schema (from billing migrations):**
- `id` (UUID PK)
- `status` (UNPAID, PAID, CANCELED)
- `expires_at` (timestamp)
- `payment_gateway_code` (external payment reference)
- All rows indexed by `status` and `expires_at` for efficient queries

## How It Works

1. **Startup:** `BillingSchedulerApplication` starts, Spring instantiates `CancelExpiredInvoicesRunner`
2. **Execution:** `CancelExpiredInvoicesRunner.run()` invokes `CancelExpiredInvoicesApplicationService.cancelExpiredInvoices()`
3. **Query:** JDBC query fetches up to 50 unpaid invoices with `expires_at <= NOW() - INTERVAL '1 day'` (with `SKIP LOCKED` for concurrent safety)
4. **Cancellation Loop:**
    - For each invoice, call Fastpay `PUT /payments/{code}/cancel`
    - On success, update invoice: `status = CANCELED`, `reason = 'Invoice expired'`
    - On failure, log error and continue
5. **Logging:** Info log with count of canceled invoices; error logs for individual failures
6. **Transaction:** All operations run in a single transaction (atomic batch)

## Common Tasks

### Adding a New Scheduled Task
1. Create `SomeOtherApplicationService` interface in `application/`
2. Implement in `infrastructure/SomeOtherApplicationServiceJDBCImpl.java` or similar
3. Create `SomeOtherRunner` extending `ApplicationRunner`
4. Register in Spring via `@Component` annotation
5. Write integration tests for the JDBC queries

### Updating Invoice Expiration Logic
1. Edit query in `CancelExpiredInvoicesApplicationServiceJDBCImpl.java` (raw SQL)
2. Update `InvoiceProjection` DTO if new columns needed
3. Test with integration tests against real database (TestContainers)

### Modifying Fastpay Integration
1. Update `FastpayPaymentAPIClient` method signature (if cancel logic changes)
2. Update `FastpayFastpayPaymentAPIClientConfig` if auth/headers change
3. Add corresponding test stub/mock for integration tests

## Testing

**Current Status:** No tests implemented yet (opportunity for contribution)

**Recommended Test Structure:**
```
src/test/java/com/algaworks/algashop/billingscheduler/
├── application/
│   └── CancelExpiredInvoicesApplicationServiceIT.java
└── infrastructure/
    ├── CancelExpiredInvoicesRunnerTest.java
    └── FastpayPaymentAPIClientIT.java
```

**Test Patterns:**
- **Unit tests** (`*Test.java`): Mock Fastpay client, test JDBC query logic with TestContainers
- **Integration tests** (`*IT.java`): Real Postgres via TestContainers, WireMock for Fastpay stubs

## Key Concepts

### Invoice Expiration
- Invoices expire 1 day after `expires_at` timestamp
- Only `UNPAID` invoices are canceled
- Once canceled, status never changes

### Batch Processing
- Processes up to 50 invoices per run (configurable limit)
- Uses `SKIP LOCKED` to avoid contention with concurrent billing updates
- Single transaction ensures atomicity

### Fastpay Integration
- Calls `PUT /payments/{paymentCode}/cancel` on Fastpay
- Expects success HTTP 2xx response
- On failure, logs error but continues (partial success acceptable)

## Dependencies

### Internal Services
- **Billing Service** — Shares database, provides invoice schema

### External Services
- **Fastpay** — Payment gateway cancellation API

### Libraries
- **Spring REST Client** — Declarative HTTP client
- **Spring JDBC** — Direct SQL execution
- **Flyway** — Database migrations (from parent billing database)
- **Lombok** — Boilerplate reduction