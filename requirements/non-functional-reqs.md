# Non-Functional Requirements

> **Document status:** Living document.
> **Owner:** IT Coordinator
> **Last reviewed:** April 2026

---

## Overview

Non-Functional requirements describe how the system must behave, not what it must do. They define the quality attributes that govern achitecture decisions, deployment choices and acceptance criteria.

---

## NFR-100 - Security

| ID | Requirement | Priority |
|---|---|---|
| NFR-101 | All API endpoints except `/api/auth/login` and `/api/auth/register` must require a valid JWT token. | P1 |
| NFR-102 | Role validation must be performed server-side on every request. Client-side role state is used for UI rendering only and is never trusted for access decisions. | P1 |
| NFR-103 | User passwords must be stored as bcrypt hashes. Plaintext passwords must never be persisted or logged. | P1 |
| NFR-104 | JWT secrets must be stored as environment variables and must not be hardcoded in source code or committed to version control. | P1 |
| NFR-105 | The API must not expose internal stack traces or exception class names in error responses sent to the client. | P2 |
| NFR-106 | All communication between the client and server must be over HTTPS in the production environment. | P1 |
| NFR-107 | Sensitive user fields — password, date of birth, phone number — must be excluded from API responses where they are not required. | P2 |

---

## NFR-200 - Reliability & Data Integrity

| ID | Requirement | Priority |
|---|---|---|
| NFR-201 | All schema changes must be managed through Liquibase migrations. Manual `ALTER TABLE` statements must not be run against the production database. | P1 |
| NFR-202 | The application must start cleanly and apply all pending migrations automatically on startup, with no manual intervention required. | P1 |
| NFR-203 | All write operations that affect multiple tables must be wrapped in a database transaction. Partial writes must not be committed. | P1 |
| NFR-204 | The unique constraint on `(user_id, poll_id, round)` in the `votes` table must be enforced at the database level, in addition to the application-layer duplicate check. | P1 |
| NFR-205 | Foreign key constraints must be enforced for all entity relationships. Orphaned records must not be possible through normal API usage. | P1 |
| NFR-206 | Attendance records must persist across page reloads and user sessions. No attendance state may be stored in browser memory or localStorage. | P1 |

---

## NFR-300 - Performance

| ID | Requirement | Priority |
|---|---|---|
| NFR-301 | API responses for standard read operations (member list, announcements, events) must complete within 500ms under normal load on the university VM. | P2 |
| NFR-302 | The attendance round view must not issue more than O(n) database queries where n is the number of users. N+1 query patterns must be identified and resolved before deployment. | P2 |
| NFR-303 | The frontend must not block the UI during API calls. All data fetching must be asynchronous with loading state indicators. | P2 |

---

## NFR-400 - Maintainability

| ID | Requirement | Priority |
|---|---|---|
| NFR-401 | All API endpoints must be documented via Swagger annotations and accessible at `/swagger-ui.html`. | P2 |
| NFR-402 | All schema changes must include a `--comment` in the Liquibase changeset describing the purpose of the change. | P2 |
| NFR-403 | Dead code — commented-out controllers, unused imports, placeholder files — must not exist in the main branch at the time of deployment. | P2 |
| NFR-404 | A new developer must be able to run the application locally within 30 minutes by following the setup instructions in `CONTRIBUTING.md`. | P2 |
| NFR-405 | The frontend service layer (`src/services/`) must contain only API client functions. React hooks (`use*`) must live in `src/hooks/`. State management must live in `src/store/`. | P2 |
| NFR-406 | Architecture decisions with long-term implications must be recorded as ADRs in `docs/decisions/` before being merged to main. | P3 |

---

## NFR-500 - Deployability

| ID | Requirement | Priority |
|---|---|---|
| NFR-501 | The application must be containerised with Docker. The backend and frontend must each have a `Dockerfile`. A `docker-compose.yml` must orchestrate the full stack including the database. | P1 |
| NFR-502 | All environment-specific configuration — database URL, credentials, JWT secret, CORS origins — must be passed via environment variables, not hardcoded. | P1 |
| NFR-503 | The application must be deployable to a Linux VM running on the university's server infrastructure without modification to the codebase. | P1 |
| NFR-504 | The database must be initialised automatically on first deploy via Liquibase. No manual SQL scripts should be required to stand up a fresh environment. | P1 |

---

## NFR-600 - Usability

| ID | Requirement | Priority |
|---|---|---|
| NFR-601 | The UI must clearly communicate the current user's role and the active meeting state at all times. | P2 |
| NFR-602 | All destructive actions — deleting a user, closing a poll, deleting a round — must require an explicit user action and must not be reversible without database access. | P2 |
| NFR-603 | Error messages returned from the API must be surfaced to the user in the UI with enough context to understand what went wrong, without exposing technical internals. | P2 |
| NFR-604 | The attendance page must be accessible only to moderators and admins. The sidebar navigation item must not appear for standard users. | P1 |
