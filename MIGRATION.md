# Migration and Hardening Notes: Next.js to Production-Ready RoomCraft Spring Boot API

## 1. Architectural Transition

The legacy full-stack Next.js/Prisma application was migrated to a dedicated backend service:
- **Runtime:** Spring Boot 3.3.0, Java 21 LTS, PostgreSQL 16.
- **Persistence:** Spring Data JPA with Flyway database migrations (`ddl-auto: validate`).
- **Documentation:** springdoc OpenAPI 2.6.0 with Swagger UI.
- **Frontend Removal:** All Next.js pages, React components, Tailwind styling, Zustand stores, Playwright tests, and Prisma artifacts were completely removed from this repository.

---

## 2. Hardening Summary by Phase

### Phase 1: Baseline & Build
- Initialized standard Spring Boot 3.3 Maven structure with Adoptium JDK 21.
- Enabled annotation processor paths for Lombok and validated that accessors (`Product.getSlug()`, `CartItem.getQuantity()`) resolve cleanly during compilation.

### Phase 2: Configuration Profiles & Fail-Fast Safeguards
- **Default Config (`application.yml`):** Removed hardcoded default database password (`change-me`).
- **Local Profile (`application-local.yml`):** Swagger UI enabled; verbose debug logging for SQL bindings, web, and security.
- **Production Profile (`application-prod.yml`):** Swagger UI and api-docs disabled; actuator health details hidden; mandatory environment variables (`DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`) with no fallback defaults.
- **Fail-Fast Validation:** Introduced `ProdConfigurationValidator` (`EnvironmentPostProcessor`) and `MissingConfigurationFailureAnalyzer` to trap missing production properties before bean initialization and render a clean, human-readable error box without stack trace dumps.

### Phase 3: Security & Access Control
- **Authentication & RBAC:** Enforced HTTP Basic authentication across endpoints.
- **Endpoint Protection Tests:**
  - Anonymous requests to `/cart` and `/orders` return `401 Unauthorized`.
  - Customer requests to `/admin/**` return `403 Forbidden`.
  - Non-owner requests for orders return `404 Not Found` (rather than `403 Forbidden`) to protect against ID enumeration.
  - Draft products are filtered out of public catalogue endpoints.
  - DTO reflection audit confirms no response entity contains `passwordHash` or plain passwords.
- **Admin Bootstrap Validation:** Refuses to bootstrap in production if `ADMIN_PASSWORD` contains fewer than 12 characters.
- **Rate Limiting:** Added Bucket4j token-bucket rate limiting (5 requests/minute per IP) on `POST /api/v1/users/register`, returning `429 Too Many Requests` when exceeded. Documented that this limiter is per-instance only and requires a shared store (e.g. Redis) for clustered deployments.

### Phase 4: Data Integrity & Concurrency Controls
- **Pessimistic Locking:** Verified `SELECT ... FOR UPDATE` row locks in `OrderService` to prevent overselling on the last unit of inventory under concurrent requests.
- **Optimistic Locking:** Mapped `ObjectOptimisticLockingFailureException` and `OptimisticLockException` to HTTP `409 Conflict` via `GlobalExceptionHandler`.
- **Business Rule Verification:**
  - Empty carts reject checkout with zero order rows generated (`422 Unprocessable Entity`).
  - Products unpublished after being added to cart reject checkout (`422 Unprocessable Entity`).
  - Unit price snapshots in `order_items` preserve the exact purchase price regardless of future catalogue price edits.
  - Carts empty atomically upon successful order creation.
- **PostgreSQL Testcontainers:** Created `OrderDataIntegrityPostgresTest` with `@Testcontainers(disabledWithoutDocker = true)` for CI execution with real PostgreSQL containers.

### Phase 5: OpenAPI Contract & Drift Detection
- Annotated all controllers (`UserController`, `ProductController`, `AdminProductController`, `CartController`, `OrderController`) with `@Operation` and `@ApiResponses` covering all valid and error HTTP status codes.
- Added `@Schema` descriptions and representative examples to all DTO fields across records and inner classes.
- Documented pagination parameters (`category`, `page`, `size`, `sort`) with SpringDoc's `@ParameterObject`.
- Exported the complete OpenAPI 3.0 specification to `docs/openapi.json`.
- Implemented `OpenApiContractTest` using MockMvc to fetch `/v3/api-docs` and assert strict equality with `docs/openapi.json`, failing the build if the live API drifts from the specification.

### Phase 6: Operations & Observability
- **Request Tracing:** Implemented `RequestIdFilter` with `Ordered.HIGHEST_PRECEDENCE` to inspect or generate `X-Request-Id`, propagate it on HTTP response headers, and bind it to SLF4J MDC (`requestId`) with automatic cleanup.
- **Multi-Stage Dockerfile:** Configured build stage using `eclipse-temurin:21-jdk-jammy` and slim runtime stage using `eclipse-temurin:21-jre-jammy` running under an unprivileged `appuser` (UID 10001) with `HEALTHCHECK`.
- **GitHub Actions CI Workflow:** Configured `.github/workflows/ci.yml` running on `ubuntu-latest` with a PostgreSQL 16 service container, executing `mvn clean verify` on push and pull requests.

---

## 3. RoomCraft Evolution & Package Relocation

- **Package Relocation:** Renamed all packages from `com.example.sharefurniture` to `com.roomcraft.api`.
- **Application Renaming:** Main entry point renamed to `RoomCraftApplication.java`.
- **Maven Artifact Renaming:** Updated `pom.xml` groupId to `com.roomcraft` and artifactId to `roomcraft-api`.
- **Database & Container Branding:** Database name and credentials updated from `sharefurniture` to `roomcraft` across `application.yml`, `docker-compose.yml`, `.env.example`, and CI workflows.
- **Metadata Alignment:** Updated Spring Boot 3 `EnvironmentPostProcessor` and `FailureAnalyzer` registration in `src/main/resources/META-INF/spring/` and `META-INF/spring.factories`.
- **Docker Compose:** Configured dual-service orchestration for `postgres:16-alpine` and `roomcraft-api` with health checks, volumes, and network isolation.
