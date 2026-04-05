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

## FR-100 - Authentication & Session Management

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
| FR-401 | The system must allow a moderator to create a named attendance round within an active meeting. | P1 | MODERATOR, ADMIN |
| FR-402 | The system must allow a moderator to check a member into a specific attendance round. | P1 | MODERATOR, ADMIN |
| FR-403 | The system must allow a moderator to check a member out of a specific attendance round. | P1 | MODERATOR, ADMIN |
| FR-404 | The system must prevent a member from being checked into the same round twice. | P1 | MODERATOR, ADMIN |
| FR-405 | The system must record `check_in_time` and `check_out_time` timestamps on every attendance record. | P1 | MODERATOR, ADMIN |
| FR-406 | The system must allow a moderator to delete an attendance round and all associated records. | P2 | MODERATOR, ADMIN |
| FR-407 | The system must allow retrieval of all attendance records for a given round. | P1 | MODERATOR, ADMIN |
| FR-408 | The system must allow retrieval of attendance history across all rounds for a given meeting. | P2 | MODERATOR, ADMIN |
| FR-409 | Attendance state must be persisted in the database — it must not reset on page reload or logout. | P1 | All |

---

## FR-500 — Polls & Voting

| ID | Requirement | Priority | Roles |
|---|---|---|---|
| FR-501 | The system must allow a moderator to create a poll with a title, description, candidate options, majority type, and electoral body count. | P1 | MODERATOR, ADMIN |
| FR-502 | The system must prevent a user from casting a vote in a poll if they do not have an active attendance record for the associated meeting. | P1 | USER (MEMBER) |
| FR-503 | The system must prevent a user from casting a vote if their `member_status` is not `MEMBER`. | P1 | USER (MEMBER) |
| FR-504 | The system must prevent a user from voting more than once per round in the same poll. | P1 | USER (MEMBER) |
| FR-505 | The system must allow a user to change their vote within the same open round. | P2 | USER (MEMBER) |
| FR-506 | The system must record which option was selected, which round the vote was cast in, and the timestamp of the vote. | P1 | USER (MEMBER) |
| FR-507 | The system must support multi-round elections progressing through `FIRST_ROUND`, `SECOND_ROUND`, and `TIE_BREAKER`. | P1 | MODERATOR, ADMIN |
| FR-508 | The system must allow a moderator to close a poll, preventing further votes. | P1 | MODERATOR, ADMIN |
| FR-509 | The system must allow an admin to delete a poll and all associated votes. | P2 | ADMIN |
| FR-510 | The system must return a descriptive error message when a vote is rejected, identifying the specific eligibility condition that failed. | P2 | USER (MEMBER) |
| FR-511 | **[PARTIAL]** The system must enforce quorum before allowing a poll to be opened. Currently `quorum_required` is stored but not enforced programmatically. | P1 | MODERATOR, ADMIN |
| FR-512 | **[PARTIAL]** The system must validate voting eligibility against the specific attendance round, not just any attendance record for the meeting. | P1 | USER (MEMBER) |

---

## FR-600 — Announcements

| ID | Requirement | Priority | Roles |
|---|---|---|---|
| FR-601 | The system must allow any authenticated user to create an announcement with a title, content, and priority level. | P1 | All |
| FR-602 | The system must automatically assign the creating user as the author of an announcement. | P1 | All |
| FR-603 | The system must return all announcements sorted by creation date, most recent first. | P1 | All |
| FR-604 | The system must allow a moderator to delete any announcement. | P2 | MODERATOR, ADMIN |
| FR-605 | The system must support an expiry date on announcements. | P2 | All |
| FR-606 | **[PARTIAL]** The system must filter out expired announcements from the `GET /api/announcements` response. Currently expiry is stored but not enforced. | P2 | All |

---

## FR-700 — Events

| ID | Requirement | Priority | Roles |
|---|---|---|---|
| FR-701 | The system must allow an authenticated user to create an event with a title, description, start time, end time, and optional location. | P1 | All |
| FR-702 | The system must allow retrieval of all events. | P1 | All |
| FR-703 | The system must allow deletion of an event. | P2 | MODERATOR, ADMIN |

---

## FR-800 — Reports

| ID | Requirement | Priority | Roles |
|---|---|---|---|
| FR-801 | The system must allow any authenticated user to submit a report to another user, with a title and content. | P1 | All |
| FR-802 | The system must record the sender, receiver, and timestamp on every report. | P1 | All |
| FR-803 | The system must allow a user to view reports sent to them (inbox) and reports they have sent. | P1 | All |
| FR-804 | The system must allow deletion of a report. | P2 | All |
