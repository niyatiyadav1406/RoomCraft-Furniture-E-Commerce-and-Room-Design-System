# RoomCraft API

Backend API for the RoomCraft store: catalogue, customer accounts, shopping cart, and order checkout.
Built with **Spring Boot 3.3**, **Java 21**, **bare-metal PostgreSQL 16**, **Flyway**, and **springdoc OpenAPI** (Swagger UI).

---

## Prerequisites

- **Java Development Kit (JDK) 21** (e.g., Eclipse Temurin 21 LTS)
- **Apache Maven 3.9+**
- **PostgreSQL 16** (locally installed and listening on `localhost:5432`)

---

## Run Locally

### 1. Provision Bare-Metal PostgreSQL 16

Ensure PostgreSQL 16 is installed and running on your host machine. Connect to `psql` as a superuser:
```sql
CREATE ROLE roomcraft WITH LOGIN PASSWORD 'secretpassword' SUPERUSER;
CREATE DATABASE roomcraft OWNER roomcraft;
CREATE DATABASE roomcraft_test OWNER roomcraft;
```

### 2. Configure Environment Variables

Copy the example environment file:
```bash
cp .env.example .env
```
Edit `.env` to supply `DB_PASSWORD` (and optional `ADMIN_EMAIL` and `ADMIN_PASSWORD`):

- **Linux / macOS (Bash / Zsh):**
  ```bash
  export $(grep -v '^#' .env | xargs)
  mvn spring-boot:run
  ```

- **Windows (PowerShell):**
  ```powershell
  $env:DB_URL="jdbc:postgresql://localhost:5432/roomcraft"
  $env:DB_USERNAME="roomcraft"
  $env:DB_PASSWORD="secretpassword"
  $env:SPRING_PROFILES_ACTIVE="local"
  mvn spring-boot:run
  ```

### 3. Verify Local Services

- **Swagger UI:** [http://localhost:8080/swagger-ui.html](http://localhost:8080/swagger-ui.html)
- **OpenAPI 3.0 JSON:** [http://localhost:8080/v3/api-docs](http://localhost:8080/v3/api-docs)
- **Actuator Health:** [http://localhost:8080/actuator/health](http://localhost:8080/actuator/health)

Flyway automatically applies database migrations from `src/main/resources/db/migration/` on startup. Hibernate runs with `ddl-auto: validate` so schemas are modified strictly through versioned migrations.

---

## Configuration Profiles

The application uses separate, explicit configuration profiles:

- **`local` (`application-local.yml`):**
  - Swagger UI and OpenAPI documentation enabled.
  - Verbose debug logging for web, security, SQL bindings, and domain services.
- **`prod` (`application-prod.yml`):**
  - Swagger UI and OpenAPI api-docs disabled (`false`).
  - Actuator health details hidden (`never`).
  - Strict fail-fast validation: `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `ADMIN_EMAIL`, and `ADMIN_PASSWORD` are strictly required with NO default values. Missing any variable produces an immediate, human-readable startup error without stack trace dumps.
  - Minimum `ADMIN_PASSWORD` length of 12 characters enforced.

---

## Environment Variables

| Name | Required in prod? | Required in local? | Example | Purpose |
|---|---|---|---|---|
| `DB_URL` | **Yes** | No (defaults to `jdbc:postgresql://localhost:5432/roomcraft`) | `jdbc:postgresql://db.prod:5432/roomcraft` | JDBC connection URL for PostgreSQL database |
| `DB_USERNAME` | **Yes** | No (defaults to `roomcraft`) | `roomcraft` | PostgreSQL database username |
| `DB_PASSWORD` | **Yes** | **Yes** (no default value) | `secretpassword` | PostgreSQL database user password |
| `PORT` | No | No (defaults to `8080`) | `8080` | Port on which the Spring Boot application listens |
| `ADMIN_EMAIL` | **Yes** | No (optional bootstrap in local) | `admin@roomcraft.com` | Email address for initial bootstrap administrator account |
| `ADMIN_PASSWORD` | **Yes** (>= 12 chars) | No (optional in local, >= 12 chars if set) | `AdminPass_2026_Secure!` | Password for bootstrap admin account (never stored in source) |
| `SPRING_PROFILES_ACTIVE` | No (defaults to `default`) | No (optional, use `local`) | `prod` or `local` | Active Spring profile controlling security, logging, and docs |

---

## Authentication & Security

- **HTTP Basic Authentication:** Protected endpoints authenticate via HTTP Basic using email and password.
- **Customer Registration:** Public endpoint at `POST /api/v1/users/register`.
- **Admin Bootstrap:** An administrative account is created on initial startup if `ADMIN_EMAIL` and `ADMIN_PASSWORD` are provided. The password must contain at least 12 characters.
- **Role-Based Access Control:**
  - Anonymous callers cannot access `/api/v1/cart/**` or `/api/v1/orders/**` (returns `401 Unauthorized`).
  - Customers (`ROLE_CUSTOMER`) cannot access `/api/v1/admin/**` (returns `403 Forbidden`).
  - Customers attempting to view another customer's order receive `404 Not Found` (rather than `403 Forbidden`) to prevent order ID enumeration.
  - DRAFT products are omitted from the public catalogue; only PUBLISHED items appear.
  - User passwords or password hashes (`passwordHash`) are never exposed in any DTO response.
- **Rate Limiting:**
  - `POST /api/v1/users/register` is protected against credential stuffing and bot abuse using Bucket4j (5 requests/minute per client IP).
  - *Note: Registration rate limiting is in-memory and per-instance only; multi-instance deployments require a distributed store such as Redis.*

---

## API Endpoints Overview

| Area | Method & Path | Access | Status Codes |
|---|---|---|---|
| Users | `POST /api/v1/users/register` | Public | 201, 400, 409, 429 |
| Users | `GET /api/v1/users/me` | Authenticated | 200, 401 |
| Products | `GET /api/v1/products?category=&page=&size=&sort=` | Public (published only) | 200 |
| Products | `GET /api/v1/products/{slug}` | Public (published only) | 200, 404 |
| Admin Products | `GET /api/v1/admin/products?page=&size=&sort=` | ADMIN | 200, 401, 403 |
| Admin Products | `POST /api/v1/admin/products` | ADMIN | 201, 400, 401, 403, 409 |
| Admin Products | `PUT /api/v1/admin/products/{id}` | ADMIN | 200, 400, 401, 403, 404, 409 |
| Admin Products | `POST /api/v1/admin/products/{id}/publish` | ADMIN | 200, 401, 403, 404 |
| Cart | `GET /api/v1/cart` | Authenticated | 200, 401 |
| Cart | `POST /api/v1/cart/items` | Authenticated | 200, 400, 401, 404, 422 |
| Cart | `PATCH /api/v1/cart/items/{productId}` | Authenticated | 200, 400, 401, 404, 422 |
| Cart | `DELETE /api/v1/cart/items/{productId}` | Authenticated | 200, 401, 404 |
| Cart | `DELETE /api/v1/cart` | Authenticated | 204, 401 |
| Orders | `POST /api/v1/orders/checkout` | Authenticated | 201, 401, 404, 409, 422 |
| Orders | `GET /api/v1/orders` | Authenticated (own orders) | 200, 401 |
| Orders | `GET /api/v1/orders/{id}` | Authenticated (own orders) | 200, 401, 404 |

All errors conform to RFC 7807 `application/problem+json`.

---

## Operations & Tracing

- **Request Tracing:** The `RequestIdFilter` inspects incoming requests for an `X-Request-Id` header, generates a UUID if absent, propagates it on the HTTP response, and attaches it to SLF4J MDC (`requestId`) for distributed log correlation.
- **Native Jar Execution:** Run the packaged application directly on bare-metal:
  ```powershell
  java -jar target/roomcraft-api-0.2.0.jar
  ```

---

## Verification & Testing

Run all unit, security, data integrity, and OpenAPI contract drift tests:
```powershell
$env:MAVEN_OPTS = "-Djavax.net.ssl.trustStoreType=WINDOWS-ROOT -Xmx512m -Xms128m"
mvn clean verify
```
The build automatically enforces that the live OpenAPI specification does not drift from `docs/openapi.json`.
