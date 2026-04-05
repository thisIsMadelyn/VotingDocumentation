# Functional Requirements

> **Document status:** Living document - update when behavior changes.
> **Owner:** IT Coordinator
> **Last reviewed:** April 2026
> **Github mapping:** requirements

---

## Overview

Functional requirements describe what the system must do. Each requirements is assigned a unique identifier (FR-XXX), a priority and the role(s) it applies to. Requirements marked **[PARTIAL]** are implemented but with known gaps documented in the Feature Scope Map.

**Prioritty levels:**
- `P1` - Must have. The system cannot go to production without this.
- `P2` - Should have. Important for correct operation but not an immediate blocker.
- `P3` - Nice to have. Improves experience but can be deferred.

---

## FR-100 - Authentacation & Session Management

| ID | Requirement | Priority | Roles |
|---|---|---|---|
| FR-101 | The system must allow a registered user to log in with a username and password and receive a JWT token. | P1 | All |
| FR-102 | The system must embed the user's role and ID in the JWT token at login time. | P1 | All |
| FR-103 | The system must reject requests to protected endpoints that do not carry a valid JWT token, returning `401 Unauthorized`. | P1 | All |
| FR-104 | The system must reject requests where the JWT is valid but the user's role is insufficient for the requested operation, returning `403 Forbidden`. | P1 | All |
| FR-105 | The system must allow a user to log out by clearing their client-side session. Server-side token invalidation is not required for the current release. | P2 | All |
| FR-106 | The system must prevent a logged-in user from accessing the login page — they must be redirected to the dashboard. | P2 | All |

---

## FR-200 — User & Membership Management

| ID | Requirement | Priority | Roles |
|---|---|---|---|
| FR-201 | The system must allow an admin to create a new user account with a username, password, email, and initial membership status. | P1 | ADMIN |
| FR-202 | The system must allow an admin to delete a user account permanently. | P1 | ADMIN |
| FR-203 | The system must allow an admin to promote a user to `MODERATOR`. | P1 | ADMIN |
| FR-204 | The system must allow an admin to demote a `MODERATOR` back to `USER`. | P1 | ADMIN |
| FR-205 | The system must allow an admin to partially update any user's profile fields. | P2 | ADMIN |
| FR-206 | The system must allow any authenticated user to view the member directory. | P1 | All |
| FR-207 | The system must allow a user to be assigned to a committee with a role title, start date, and optional end date. | P2 | ADMIN |
| FR-208 | The system must store `member_status` separately from `login_role` to support users who are authenticated but not yet full voting members. | P1 | All |

---

## FR-300 — Meeting Management

| ID | Requirement | Priority | Roles |
|---|---|---|---|
| FR-301 | The system must allow a moderator to create a General Meeting with a date and quorum requirement. | P1 | MODERATOR, ADMIN |
| FR-302 | The system must allow exactly one General Meeting to be active at any given time. | P1 | MODERATOR, ADMIN |
| FR-303 | The system must expose an endpoint that returns the currently active General Meeting. | P1 | All |
| FR-304 | The system must allow an admin to delete a General Meeting. | P2 | ADMIN |
| FR-305 | The system must return `404 Not Found` when no active meeting exists, rather than returning an empty response. | P1 | All |

---

## FR-400 — Attendance Management

| ID | Requirement | Priority | Roles |
|---|---|---|---|








