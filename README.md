# User Management Service

Enterprise-grade microservice for user lifecycle management, built with Java 21 + Spring Boot 3.2.

---

## Quick Start

```bash
# Clone and start everything with one command
git clone <repo-url>
cd user-management-service
docker-compose up -d

# Service available at:
#   API:         http://localhost:8080/api/v1/users
#   Swagger UI:  http://localhost:8080/swagger-ui.html
#   RabbitMQ UI: http://localhost:15672 (guest/guest)
#   Keycloak:    http://localhost:9090  (admin/admin)
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    user-management-service                      │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐     │
│  │ Controller   │ → │   Service    │ → │   Repository     │     │
│  │ (REST/HTTP)  │   │ (Business    │   │ (Spring Data JPA)│     │
│  │              │   │  Logic)      │   │                  │     │
│  └──────────────┘   └──────────────┘   └──────────────────┘     │
│         │                  │                      │             │
│    DTOs + Validation   Events (async)        PostgreSQL         │
│         │                  │                                    │
│    GlobalException     RabbitMQ                                 │
│    Handler                                                      │
└─────────────────────────────────────────────────────────────────┘
         │                  │
    Keycloak (IAM)     ums.events exchange
    JWT validation     → ums.user.created queue
```

**Layer responsibilities:**

- **Controller** — HTTP concerns only: routing, request/response mapping, auth annotations, Swagger docs
- **Service** — all business logic: validation, state transitions, event publishing
- **Repository** — data access: custom JPQL queries, pagination, soft-delete filtering
- **Mapper** — explicit DTO ↔ Entity conversion via MapStruct (no implicit field leakage)

---

## Technical Choices & Trade-offs

### Database: PostgreSQL

**Why PostgreSQL over MongoDB:**

The domain has clear relational structure (users, roles) with strict constraints (unique email, unique CF, referential integrity on user_roles). PostgreSQL gives:
- **ACID transactions** — critical when checking uniqueness and inserting atomically
- **Partial unique indexes** — `uq_users_cf_active` and `uq_users_username_active` enforce uniqueness only among non-deleted users, avoiding conflicts with soft-deleted records
- **UUID PK** — distributed-system ready; no coordination required for ID generation
- Native `gen_random_uuid()` for zero-dependency UUID generation

MongoDB would have been a better choice if the user schema was highly variable or document-oriented. It is not.

### Identity & Access Management: Keycloak

The service acts as an **OAuth2 Resource Server** — it trusts and validates JWTs issued by Keycloak but never issues tokens itself. This follows the standard enterprise microservice pattern:

- Auth logic stays in Keycloak, not in application code
- JWTs carry Keycloak realm roles (`ADMIN`, `MANAGER`, `VIEWER`) in the `realm_access.roles` claim
- A custom `JwtAuthenticationConverter` maps these to Spring Security `GrantedAuthority` objects
- `@PreAuthorize` annotations enforce RBAC at the method level

**Three security roles:**

| Role    | Permissions |
|---------|-------------|
| ADMIN   | Full CRUD + disable/delete |
| MANAGER | Create, update, read (unmasked) |
| VIEWER  | Read-only; email and CF are masked |

**Domain roles vs security roles:** `OWNER`, `OPERATOR`, `MAINTAINER`, `DEVELOPER`, `REPORTER` are *application-level* roles stored in the database and assigned to users. They are distinct from the security roles used for API authentication.

### Async Events: RabbitMQ with Topic Exchange

When a user is created, a `UserCreatedEvent` is published to the `ums.events` topic exchange with routing key `user.created`. Downstream services (notification, audit, Keycloak sync) subscribe independently.

**Why topic exchange over direct:**
- Wildcards (`user.#`) allow consumers to subscribe to all future user events without topology changes
- Adding `user.updated`, `user.deleted` events in the future requires no changes to the exchange

**Dead-letter queue (`ums.user.created.dlq`):** failed messages are routed here for inspection rather than being lost.

**Resilience design:** event publishing failure does NOT roll back user creation. The user record is always saved first. In production this pattern would be complemented by the [Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html) for guaranteed delivery.

### Schema Management: Flyway

Flyway owns all DDL. Hibernate is configured with `ddl-auto: validate` — it only validates the schema, never modifies it. This is the correct production approach:
- Migrations are versioned and repeatable
- Schema changes are tracked in VCS
- Safe for blue/green deployments

### Soft Delete

Users are never physically deleted. `DELETE /users/{id}` sets `status = 'DELETED'`. Reasons:
- Audit trail preserved
- Foreign key references remain valid
- Reversible by ops team directly in DB if needed

All queries use `WHERE status != 'DELETED'` to exclude them transparently.

### Correlation ID

Every request receives an `X-Correlation-ID` (generated if not provided by the caller). It is stored in the MDC and included in all log lines for that request. This enables end-to-end request tracing across services and is essential in a microservices architecture.

---

## API Reference

Full interactive documentation: **http://localhost:8080/swagger-ui.html**

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET    | `/api/v1/users` | ADMIN, MANAGER, VIEWER | List users (paginated + filterable) |
| GET    | `/api/v1/users/{id}` | ADMIN, MANAGER, VIEWER | Get user by ID |
| POST   | `/api/v1/users` | ADMIN, MANAGER | Create user |
| PATCH  | `/api/v1/users/{id}` | ADMIN, MANAGER | Update user data/roles |
| PATCH  | `/api/v1/users/{id}/disable` | ADMIN | Disable user |
| DELETE | `/api/v1/users/{id}` | ADMIN | Soft-delete user |

### Authentication

```bash
# 1. Get a token from Keycloak
TOKEN=$(curl -s -X POST http://localhost:9090/realms/ums/protocol/openid-connect/token \
  -d "client_id=ums-client&client_secret=ums-client-secret&grant_type=password" \
  -d "username=admin-user&password=admin123" | jq -r .access_token)

# 2. Use the token
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/v1/users
```

### Example: Create a User

```bash
curl -X POST http://localhost:8080/api/v1/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "mario.rossi",
    "email": "mario.rossi@example.com",
    "codiceFiscale": "RSSMRA85M01H501Z",
    "nome": "Mario",
    "cognome": "Rossi",
    "roles": ["DEVELOPER", "REPORTER"]
  }'
```

### Query Parameters for List

| Parameter | Type | Description |
|-----------|------|-------------|
| `status`  | `ACTIVE` \| `DISABLED` \| `DELETED`| Filter by status |
| `search`  | string | Free-text search on username, email, nome, cognome |
| `page`    | int (default: 0) | Page number |
| `size`    | int (default: 20, max: 100) | Page size |
| `sortBy`  | string (default: `createdAt`) | Sort field |
| `direction` | `ASC` \| `DESC (default: DESC)` | Sort direction |

---

## Build & Run

### Prerequisites
- Docker + Docker Compose
- Java 21 (for local development without Docker)

### Full stack with Docker
```bash
docker-compose up -d
docker-compose logs -f app   # Follow logs
docker-compose down          # Stop
```

### Local development (without Docker app container)
```bash
# Start only infrastructure
docker-compose up -d postgres rabbitmq keycloak

# Run the app
./mvnw spring-boot:run
```

### Tests
```bash
# Unit tests only
./mvnw test -Dtest="*Test"

# Integration tests (requires Docker for Testcontainers)
./mvnw verify

# Full suite
./mvnw verify
```

---

## Project Structure

```
src/main/java/com/intesi/ums/
├── config/
│   ├── SecurityConfig.java        # OAuth2 Resource Server, Keycloak JWT converter
│   ├── RabbitMQConfig.java        # Exchange, queues, DLQ topology
│   ├── OpenApiConfig.java         # Swagger/OpenAPI metadata
│   ├── AuditConfig.java           # Spring Data Auditing (createdBy/updatedBy)
│   └── CorrelationIdFilter.java   # X-Correlation-ID MDC injection
├── controller/
│   └── UserController.java        # REST endpoints with RBAC annotations
├── domain/
│   ├── User.java                  # JPA entity with auditing
│   ├── UserStatus.java            # ACTIVE | DISABLED | DELETED
│   └── ApplicationRole.java       # OWNER | OPERATOR | MAINTAINER | DEVELOPER | REPORTER
├── dto/
│   ├── CreateUserRequest.java     # Validated creation payload
│   ├── UpdateUserRequest.java     # Partial update payload
│   ├── UserResponse.java          # Response DTO with masking helpers
│   ├── PagedResponse.java         # Generic pagination wrapper
│   └── ErrorResponse.java        # Standardised error structure
├── exception/
│   ├── UserNotFoundException.java
│   ├── DuplicateResourceException.java
│   ├── IllegalStateTransitionException.java
│   └── GlobalExceptionHandler.java  # Centralised error mapping
├── mapper/
│   └── UserMapper.java            # MapStruct: full + masked response variants
├── messaging/
│   ├── UserCreatedEvent.java      # Domain event record
│   └── UserEventPublisher.java    # RabbitMQ publisher
├── repository/
│   └── UserRepository.java        # Custom JPQL queries, soft-delete aware
├── service/
│   ├── UserService.java           # Interface
│   └── UserServiceImpl.java       # Business logic implementation
└── validation/
    ├── ValidCodiceFiscale.java    # Custom annotation
    └── CodiceFiscaleValidator.java # Full Italian CF checksum validation
```

---

## Possible Improvements (Production Roadmap)

- **Outbox Pattern** for guaranteed at-least-once event delivery to RabbitMQ
- **Distributed caching** (Redis) for frequently accessed user data
- **Rate limiting** on the POST endpoint to prevent abuse
- **Keycloak user sync** — create a corresponding Keycloak user account on `POST /users`
- **Pagination cursor-based** instead of offset for large datasets
- **Observability** — Micrometer + Prometheus metrics, OpenTelemetry traces
