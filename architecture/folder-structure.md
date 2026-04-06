# Folder Structure

> **Document status:** Living document — update when new directories are added.  
> **Owner:** IT Coordinator  
> **Last reviewed:** April 2026  
> **GitHub mapping:** `/architecture`

---

## Overview

the project is split into two separate repositories - one for the backend (Spring Boot) and one for the frontend (React). this document covers both. Conventions documented here are enforced requirements, not suggestions - see NFR-405.

---

## Backend - Spring Boot

```
votingsystem/
├── src/
│   ├── main/
│   │   ├── java/com/eestec/votingsystem/
│   │   │   ├── controller/         # REST controllers — HTTP mapping only, no business logic
│   │   │   ├── dto/                # Data Transfer Objects — request/response shapes
│   │   │   ├── entity/             # JPA entities — one file per DB table
│   │   │   ├── enums/              # Enum definitions (LoginRole, AttendanceMethod, etc.)
│   │   │   ├── exceptions/         # Custom exception classes
│   │   │   ├── repository/         # Spring Data JPA repositories
│   │   │   ├── service/            # Business logic — one service per domain
│   │   │   ├── security/           # Spring Security config, JWT filter, UserDetailsService
│   │   │   └── VotingApplication.java   # Spring Boot entry point
│   │   └── resources/
│   │       ├── db/
│   │       │   └── changelog/      # Liquibase migration files
│   │       │       ├── db.changelog-master.sql
│   │       │       ├── 001-initial-schema.sql
│   │       │       ├── 002-add-is-active-meetings.sql
│   │       │       ├── 003-add-attendance-round.sql
│   │       │       ├── 004-add-round-id-to-attendance.sql
│   │       │       └── 005-add-created-by-announcements.sql
│   │       └── application.properties   # Environment configuration
│   └── test/
│       └── java/com/eestec/votingsystem/
│           └── ...                 # Integration and unit tests
├── pom.xml
└── Dockerfile                      # To be added before deployment
```

### Backend Layer Responsibilities

| Layer | Package | Rule | 
|---|---|---|
| Controller | `controller/` | Maps HTTP to service calls. No business logic. No direct repository access. |
| Service | `service/` | All business logic lives here. Owns transactions (`@Transactional`). |
| Repository | `repository/` | Spring Data JPA interfaces only. Custom queries via `@Query` when method name parsing is insufficient. |
| Entity | `entity/` | JPA-mapped classes. No business logic. `@JsonIgnoreProperties` on all bidirectional relationships. |
| DTO | `dto/` | Request and response shapes. Decouples the API contract from the entity model. |
| Security | `security/` | JWT filter, Spring Security configuration, `UserDetailsService` implementation. Never modified for feature work. |
| Enums | `enums/` | All enum types used across the application. Changes here require a corresponding Liquibase migration if the enum is persisted. |
| Exceptions | `exceptions/` | Custom `RuntimeException` subclasses. All must carry `@ResponseStatus` to prevent Spring from defaulting to `500`. |

### Backend Conventions
 
- One controller per domain (`AttendanceCheckController`, `PollsController`, etc.). Do not add unrelated endpoints to an existing controller.
- One service per domain. Cross-domain operations go in the service that owns the primary entity, calling other services as dependencies.
- Never call a repository directly from a controller.
- Every new DB column requires a new numbered Liquibase changeset. Never modify an existing changeset that has already been applied.
- Migration files are numbered sequentially: `006-description.sql`, `007-description.sql`, etc. The description must match the `--comment` in the changeset.

---

## Frontend — React + Vite
 
```
voting_ui_react_demo/
├── public/
│   └── vite.svg
├── src/
│   ├── assets/                     # Static assets (logos, images)
│   ├── components/
│   │   ├── layout/                 # App shell components (Sidebar, TopBar, Layout, MainPanel)
│   │   └── ui/                     # Reusable atomic components (Card, Icon, RoleTag, SearchBar, etc.)
│   ├── features/
│   │   ├── attendance/
│   │   │   └── comps/              # Attendance-specific components
│   │   │       ├── ActiveMeetingTab.jsx
│   │   │       ├── ActiveMeetingTab.module.css
│   │   │       ├── AttendanceHistoryTab.jsx
│   │   │       ├── AttendanceHistoryTab.module.css
│   │   │       ├── AttendanceRoundCard.jsx
│   │   │       └── AttendanceRoundCard.module.css
│   │   └── dashboard/
│   │       ├── comps/              # Dashboard-specific components
│   │       └── data/               # Mock data for dashboard (development only)
│   ├── hooks/                      # All custom React hooks (use*.js)
│   │   ├── useAttendance.js
│   │   ├── useAnnouncements.js
│   │   ├── useEvents.js
│   │   ├── useMembers.js
│   │   ├── usePolls.js
│   │   ├── useReports.js
│   │   ├── useUsers.js
│   │   ├── useVoting.js
│   │   └── useDismissed.js
│   ├── pages/                      # Route-level page components (one per route)
│   │   ├── AttendancePage.jsx
│   │   ├── DashBoardPage.jsx
│   │   ├── EventPage.jsx
│   │   ├── LoginPage.jsx
│   │   ├── MembersPage.jsx
│   │   ├── AnnouncementsPage.jsx
│   │   ├── PollsPage.jsx
│   │   └── ReportsPage.jsx
│   ├── router/
│   │   ├── AppRouter.jsx           # Route definitions and layout composition
│   │   ├── ProtectedRoute.jsx      # Auth guard — redirects unauthenticated users to login
│   │   └── Routes.js               # Route path constants
│   ├── services/                   # API client functions only — no hooks, no state
│   │   ├── AxiosClient.js          # Axios instance with JWT interceptor
│   │   ├── AttendanceApi.js
│   │   ├── AuthApi.js
│   │   ├── EventsApi.js
│   │   ├── MeetingApi.js
│   │   ├── MembersApi.js
│   │   ├── PollsApi.js
│   │   ├── ReportsApi.js
│   │   ├── UsersApi.js
│   │   ├── VotingApi.js
│   │   ├── AnnouncementsApi.js
│   │   ├── weatherService.js
│   │   └── services_readme.md      # Documents the services layer conventions
│   ├── store/
│   │   └── AuthStore.js            # Zustand auth state (isAuthenticated, user, token)
│   ├── styles/
│   │   ├── variables.css           # CSS custom properties — design tokens
│   │   ├── global.css              # Global resets and base styles
│   │   └── utils.css               # Utility classes
│   ├── App.jsx
│   ├── main.jsx
│   └── index.css
├── index.html
├── vite.config.js
├── eslint.config.js
├── package.json
└── Dockerfile                      # To be added before deployment
```

### Frontend Layer Responsibilities

| Directory | Rule |
|---|---|
| `services/` | API functions only. Pure async functions that call Axios and return data. No `useState`, no `useEffect`, no React imports. |
| `hooks/` | All `use*` files. TanStack Query mutations and queries. May import from `services/` but never from `pages/` or `features/`. |
| `store/` | Global state only. Currently contains `AuthStore.js` (Zustand). Do not add feature state here — feature state belongs in hooks or local component state. |
| `components/ui/` | Atomic, reusable components with no business logic and no direct API calls. Props only. |
| `components/layout/` | App shell. The sidebar, topbar, and main panel. Modified only when the shell itself changes — not for feature work. |
| `features/` | Feature-specific components that are too complex for `components/ui/` but are not route-level pages. Organised by domain. |
| `pages/` | One file per route. Composes components and hooks. No inline business logic. |
| `styles/` | Design tokens and global styles only. Component styles live in `.module.css` files co-located with their component. |
 
### Frontend Conventions
 
- Every component has a co-located `.module.css` file. No inline styles except for dynamic values that cannot be expressed as CSS variables.
- All CSS values use the design token variables defined in `styles/variables.css`. No hardcoded hex values, no hardcoded spacing values.
- Route path strings are defined as constants in `router/Routes.js`. Never hardcode a path string in a component.
- A page component (`pages/`) must never be imported by another component — only by `AppRouter.jsx`.
- A feature component (`features/`) must never import from `pages/`.
- `AxiosClient.js` is the only file that configures Axios. All API functions import the client from this file — they never create their own Axios instance.
- New API domains get a new `*Api.js` file in `services/` and a corresponding `use*.js` file in `hooks/`.

 ### Adding a New Feature — Checklist
 
When adding a new feature end-to-end:
 
**Backend:**
- [ ] New Liquibase changeset if schema changes are needed
- [ ] New entity in `entity/` with `@JsonIgnoreProperties` on relationships
- [ ] New repository in `repository/`
- [ ] New service in `service/` with `@Transactional` on write methods
- [ ] New controller in `controller/` with `@Operation` Swagger annotations
- [ ] `@ResponseStatus` on any new exception classes
 
**Frontend:**
- [ ] New `*Api.js` in `services/` for API calls
- [ ] New `use*.js` in `hooks/` for TanStack Query wrappers
- [ ] New route constant in `router/Routes.js` if a new page is needed
- [ ] New page in `pages/` if a new route is needed
- [ ] New feature folder in `features/` for complex domain components
- [ ] New route in `AppRouter.jsx` wrapped in `ProtectedRoute` if auth is required
- [ ] New nav item in `Sidebar.jsx` if the feature needs a sidebar link
- [ ] CSS variables used throughout — no hardcoded values
 
