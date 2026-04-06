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

## 4.5 DAtabase Architecture

The MySQL database is the single source of truth for all application state. It is accessed exclusively through th Spring DAta JPA repository layer - no direct SQL query from the application code outside of Liquibase migrations.

**Schema management:** All schema changes are versioned in Liquibase changelogs located at `src/main/resources/db/changelog`. Liquibase applies pending migrations automatically son application startup. The `DATABASECHANGELOG` table tracls which migrations have been applied.




















 
