# System Architecture

> **Document status:** Living document - update before each deployment.
> **Owner:** IT Coordinator
> **Last reviewed:** April 2026
> **GitHub mapping:** `architecture/` folder

 ---

## 4.1 Overview

The system is a three-tier web application consisting ofa React frontend, A Spring Boot API and a MySQL database. All three tiers are deployed on a single university-managed virtual machine running Linux. In the production environment, the tiers are isolated as Docker containers orchestrated by Docker Compose.

```
┌──────────────────────────────────────────────────────┐
│                  University VM (Linux)               │
│                                                      │
│  ┌──────────────┐     ┌──────────────┐    ┌────────┐ │
│  │   Frontend   │     │   Backend    │    │  MySQL │ │
│  │   React +    │ ->  │ Spring Boot  │ -> │  :3306 │ │
│  │    Vite      │     │    :8080     │    │        │ │
│  │   :5173      │     │              │    │        │ │
│  └──────────────┘     └──────────────┘    └────────┘ │
│                                                      │
└──────────────────────────────────────────────────────┘
```

All inter-service communication is internal to tthe VM. External users access the frontend only. The backend API is not direclty exposed to the public internet - all API calls are proxied through thee frontend's Vite dev server in development and through a reverse proxy (to be configured) in production.

---

## 4.2 Technology Stack

### Backend

| Component | Technology | Version |
|---|---|---|
| Language | Java | 21 |
| Framework | Spring Boot | 4.0.1 |
| Security | Spring Security + JWT (jjwt) | 0.12.5 |
| Persistence | Spring Data JPA + Hibernate | Managed by Spring Boot |
| Database migrations | Liquibase | 4.29.2 |
| API documentation | SpringDoc OpenAPI (Swagger UI) | 3.0.0 |
| Build tool | Maven | — |
| Boilerplate reduction | Lombok | — |

### Frontend

| Component | Technology | Version |
|---|---|---|
| Language | JavaScript (ES2022+) | — |
| Framework | React | 18+ |
| Build tool | Vite | — |
| HTTP client | Axios | — |
| State management | Zustand (`AuthStore`) | — |
| Server state | TanStack Query (React Query) | — |
| Routing | React Router v6 | — |
| Styling | CSS Modules + CSS custom properties | — |

### Infrastructure 

| Component | Technology |
|---|---|
| Containerisation | Docker + Docker Compose |
| Database | MySQL 8 |
| Host | University VM (Linux) |
| Reverse proxy | To be configured — Nginx recommended |

---

## 4.3 Request Lifecycle

A typical authenticated request follows this path:

```
Browser
  │
  │  HTTP request with Authorization: Bearer <token>
  ▼
React Frontend (:5173)
  │
  │  Axios interceptor attaches JWT from localStorage
  │  to every outbound request automatically
  ▼
Spring Boot Backend (:8080)
  │
  ├─ Spring Security Filter Chain
  │    Extracts and validates JWT
  │    Populates SecurityContextHolder with authenticated user
  │    Rejects invalid/missing tokens with 401
  │
  ├─ Controller Layer
  │    Maps HTTP method + path to handler
  │    Deserialises request body
  │    Delegates to service layer
  │
  ├─ Service Layer
  │    Enforces business rules
  │    Validates role access (ModeratorValidatorService)
  │    Validates domain constraints (VotingValidationService)
  │    Manages transactions (@Transactional)
  │
  ├─ Repository Layer (Spring Data JPA)
  │    Executes queries against MySQL
  │    Returns entities or throws exceptions
  │
  └─ Response
       Serialised to JSON by Jackson
       @JsonIgnoreProperties prevents circular references
       Returned to frontend with appropriate HTTP status
```

---

## 4.4 Authentication Flow

```
User submits credentials
  │
  ▼
POST /api/auth/login
  │
  ├─ UserDetailsService loads user from DB by username
  ├─ BCrypt verifies password hash
  ├─ JWT generated with claims: username, role, userId
  └─ Token returned to client
        │
        ▼
     Frontend stores token + user object in localStorage
        │
        ▼
     Axios interceptor attaches token to all subsequent requests
        │
        ▼
     Spring Security JwtAuthenticationFilter validates token on each request
     SecurityContextHolder populated with authenticated principal
```

**Token conteents:**

| Claim | Value |
|---|---|
| `sub` | Username |
| `role` | `ADMIN`, `MODERATOR`, or `USER` |
| `userId` | Database ID of the user |
| `iat` | Issued at timestamp |
| `exp` | Expiry timestamp |

**Token lifecycle:** Tokens are not invalidated server-side on logout or role change. Logout clears localStorage only. A demoted user's token remains valid until natural expiry. See NFR-101 and the Role Escalation section in User Roles for mitigation guidance.

---

## 4.5 Database Architecture

The MySQL database is the single source of truth for all application state. It is accessed exclusively through th Spring DAta JPA repository layer - no direct SQL query from the application code outside of Liquibase migrations.

**Schema management:** All schema changes are versioned in Liquibase changelogs located at `src/main/resources/db/changelog`. Liquibase applies pending migrations automatically son application startup. The `DATABASECHANGELOG` table tracls which migrations have been applied.

**Connections:** the backend to MySQL on `localhost:3306` in the production VM environment. Connection credentials are passed via environment variables - they are never hardcoded.

**Key design decisions:**

- `is_active` on `general_meetings` — application-level flag for active meeting detection. At most one meeting should be active at a time. DB-level enforcement (unique partial index or trigger) is a planned improvement.
- `round_id` on `attendance_check` — nullable FK added to support multi-round attendance. Existing records without a round assignment remain valid.
- `UNIQUE(user_id, poll_id, round)` on `votes` — database-level constraint preventing duplicate votes, enforced independently of the application-layer duplicate check in `VotingValidationService`.
- `created_by` on `announcements` — nullable FK to `users`. Announcements created before this column was added have a null author — this is expected and handled gracefully in the UI.

---

## 4.6 Cross-Cutting Concerns

### Error Handling 

The backend uses a mix of controller-level try/catch and Spring's default exception handling. the follwing HTTP status codes are in use:

| Status | When used |
|---|---|
| `200 OK` | Successful read or update |
| `201 Created` | Successful resource creation |
| `204 No Content` | Successful deletion |
| `400 Bad Request` | Validation failure, business rule violation |
| `401 Unauthorized` | Missing or invalid JWT |
| `403 Forbidden` | Valid JWT but insufficient role |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Duplicate resource (e.g. already checked in) |
| `500 Internal Server Error` | Unhandled exception — should not reach production |

> **Note:** A global `@ControllerAdvice` exception handler is a planned improvement. Currently, 500 responses may leak internal exception messages in some edge cases.
 
### CORS
 
CORS is configured on `PollsController` with `@CrossOrigin(origins = "http://localhost:5317")`. In production, CORS must be configured globally via `WebMvcConfigurer` to cover all controllers and to reflect the production domain. This is a deployment blocker.
 
### Serialisation
 
Jackson is used for JSON serialisation. Circular reference prevention is handled via `@JsonIgnoreProperties` on entity relationship fields. The `password` field on `Users` is excluded from all responses via `@JsonIgnoreProperties`.

---

4.7 Deployment Architecture

### Development Environment

```
Developer machine
├── Backend: mvn spring-boot:run → :8080
├── Frontend: npm run dev → :5173
└── MySQL: local instance → :3306
```

### Production Environment (Target)
 
```
University VM
├── Docker container: spring-boot-app → :8080 (internal)
├── Docker container: react-app (served by Nginx) → :80/:443
├── Docker container: mysql → :3306 (internal only)
└── Nginx reverse proxy → routes external traffic to containers
```

Docker Compose configuration is pending. The following services must be defined:
 
- `backend` — Spring Boot JAR, environment variables for DB and JWT
- `frontend` — React build served by Nginx
- `db` — MySQL 8 with volume mount for data persistence
- Shared Docker network for internal service communication

### Environment Variables Required for Production
 
| Variable | Used by | Description |
|---|---|---|
| `DB_URL` | Backend | JDBC connection string |
| `DB_USERNAME` | Backend | MySQL username |
| `DB_PASSWORD` | Backend | MySQL password |
| `JWT_SECRET` | Backend | Secret key for JWT signing |
| `JWT_EXPIRATION` | Backend | Token expiry in milliseconds |
| `CORS_ALLOWED_ORIGINS` | Backend | Comma-separated list of allowed origins |

> **These variables must never appear in source code or be committed to version control.** Use a `.env` file locally (gitignored) and server-level environment configuration on the VM.

---

## 4.8 Known Architectue Limitations

| Limitation | Risk | Planned resolution |
|---|---|---|
| No server-side token invalidation | Demoted users retain access until token expiry | Token blacklist or short expiry window |
| CORS configured per-controller | Some controllers may be unreachable in production | Global CORS configuration via `WebMvcConfigurer` |
| No global exception handler | 500s may leak internal details | `@ControllerAdvice` with standardised error response DTO |
| Single active meeting not DB-enforced | Multiple active meetings possible if constraint is bypassed | Unique partial index on `general_meetings(is_active)` where `is_active = true` |
| Attendance validation not round-aware | Users checked into Round 1 and out may still pass vote gate | Rewrite `validateVoteEligibility` to filter by most recent round |
| Environment variables not externalised | JWT secret and DB credentials are hardcoded in current codebase | Must be resolved before any deployment |
| Docker configuration not written | Cannot deploy to VM | Write `Dockerfile` for backend and frontend, write `docker-compose.yml` |





 
