# 1. Overview & Scope 

> **Document Status:** Living Document - update on every sprint boundary.
> **Owner:** IT Coordinator
> **Last Reviewed:** April 2026

---

## 1.1 Executive Summary 

The EESTEEC LC Thessaloniki Voting & Management System is an internal web platform pupose-built for the operational needs of the Local Committe. It replaces a fragmented collection of spreadsheets, massaging theads and manual processes with a single, role-aware dashboard deployed on a virtualized server infrastructure.

The Platform servers three core functions:

- **Identity & Membership Management** - a single source of truth for membe records, roles and committee assignemts, stored in a relational database owned and operated by the LC.
- **Meeting operations** - structured attendance tracking with multi-round check-in support, directly gating voting eligibility for any given session.
- **Democratic Process** - a full voting pipeline covering poll creation, multi-round elections, majority-type configuration and result resolution, with cryptographic session integrity via JWT.

The system is not a generic tool. Every design decision - the attendance-to-vote gate, the 'ElectionRound' enum, the committee vote weight - reflects a deliberate mapping to how LC Thessaliniki actually operates.

---

## 1.2 Problem Statementt

### Current State

Prior to this system, the LC operated with the following pain points:

| Area | Problem |
|---|---|
| Member data | Scattered across Google Drive, Discord Server and individual board members' devices |
| Attendance | Tracked manually per meeting with no audit trail |
| Voting | Conducted verbally or via informal polls with no verifiable record |
| Access control | No technical enforcement of role boundaries - board decisions relied on trust |
| Reporting | No structured channel for member-to-board communication |

These conditions introduced risk at every General Meeting: disputed vote counts, missing quorum data and no recoverablerecord of decision made.

### Desired State

A single deployed application that enforces the following guarantees:

- Member records are persisted in a structured elational database, accessible only to authenticated users with appropriate roles.
- Attendance at a General Meeting is recordedper session and per round, with check-in and check-out timestamps. Voting eligibility is derived programmatically from attendace state - not from manual verification.
- Voting is conducted through a controlled pipeline. A vote cannot cannot be cast by a user who is not checked-in, not a full 'MEMBER', or who has already voted in the current round. These constrains are enforced at the service layer, not the UI.
- All destructive or privileged operations - poll creation, user promotion, meeting management - require an authenticated 'MODERATOR' or 'ADMIN' role, validated server-side on every request.
- The system runson infrastructure owned by the provider, keeping member data within institutional boundaries.

---

## 1.3 Target Audience & User Personas

The sustem recognises four distinct user classes, mapped to Spring Security roles and enforced via JWT claims:

### `ADMIN`
Full system access. Responsible for user lifecycle management - creating reports, promoting members to moderator and deleting records. Can delete polls and meetings. There is typically one admin per board cycle.

### `MODERATOR`
Operational access. Creates and manages General Meetings, opens and closes polls, manages attendance rounds and creates announcements, events and reports. Cannot delete users or promote other users. This is the primary role for active board members.

### `USER` - Full Member (`member_status: MEMBER`)
Can log in, view the dashboard, cast votes in active polls during meetings they are checked into. Voting eleigibility is enforced - a 'MEMBER' who is not checked in cannot vote.

### `USER`` - Junior Member (`member_status: JUNIOR`)
Can log in and view the dashboard. Cannot cast votes. This is the entry level status for new members pending full membership approval.

> **Note:** Alumni accounts exist in the system (`member_status:ALUMNI`) for historical recoed purpose. they carry no voting rights.

---

## 1.4 Project Scope

### In Scope

| Domain | Capabilities |
|---|---|
| Authentication | JWT-based login and session management. Role assignment at login. token validation on every protected request. |
| User management | CRUD operations on user accounts. Role promotion and demotion by admins. Committee membership tracking with temporal validity (start/end dates). |
| Meeting Management | General Meeting creation and deletion. Active meeting detection (`is_active` flag) |
| Attendance | Multi-round attendace check-in and check-outper meeting. Round creation and deletion by moderators. Attendance state used to gate voting eligibility. |
| Polls & Voting | Poll creation with candidate options, majority type and electoral body count. Multi-round election support (`FIRSST_ROUND`, `SECOND_ROUND`, `TIE_BREAKER`). Poll lifecycle management (open -> closed -> winner declared). |
| Announcements | Organization-wide announcents with priority levels and expiry dates. Authorship tracked to the creating user. |
| Events | Event creation and listing. Standalone - not tied to meetings. |
| Reports | Board-to-Board report submition. Sender and reciever are tracked. |
| Access control | Role-based endpoint protection at the controller and service layer. Moderator and admin access validated on every privileged operation. |

### Out of Scope

The following explicity not part of this system's current scope:

- Email or push notifications of any kind.
- Public-facing pages - the entire platform requires authentication.
- Mobile native applications - the frontend is a responsive web application only.
- Integration with external EESTEC systems.
- Payment or membership fee processing.
- Real-time feeatures (WebSockets, live vote counts during an open poll).

---

## 1.5 Success Criteria

The system is considered production ready whe nthe following conditions are met:

1. General Meeting can be created, have attendance rounds opened, members checked in and out per round and poll run to conclusion without any manual database intervention.
2. A vote cast by an ineligible user 9not checked in, wrng membership status, already voted) is rejected at the API layer with a descriptive error, regardless of what the frontend sends.
3. All privileged operations return '403 Forbidden' when called without a valid moderator or admin toekn.
4. the application starts cleanly on the providers VM with Liquibase applying all schema migrations automatically.
5. A new developer can clone the repository, follow the setup instructions and have a running local environment within 30 minutes.

---

## a.6 Constrains & Risks

| Constraint / Risk | Impact | Mitigation |
|---|---|---|
| Deployed on provider Infrastructure | Limited control over uptime, network configuration and OS-level dependencies | Docker containerisation isolates the application from host environment specifics |
| No dedicated DBA | SChema migrations managed by developers | Liquibase enforces versioned, auditable schema changes - no manual `ALTER TABLE` IN production |
| Student Volunteer maintainers | Knowledge transfer risk between board cycles | "Wiki First" documentation policy - all decisions recorded before deployment |
| Single active meeting constraint | 'is_active' flag currently enforced at application level only | Future: add a DB-level check constraint or trigger to guarantee at most one active meeting |
| Attendance querry does not account for rounds | 'validatevoteEligibility' checks first attendance record, not round-specific | Known limitation - tobe addressed before second round voting is used in production |

---

## 1.7 GitHub Mapping

| Documentation section | GitHub equivalent |
|---|---|
| This document (Project Brief) | `project-brief.md` |
| Functional Requirements | Issues & Milestones |
| API Reference | `docs/openai.yaml` or Swagger UI at `/swagger-ui.html` |
| Data Model | `db/changelog/` (Liquibase) |
| Sprint Log | `CHANGELOG.md` |
| Architecture Decisions | `docs/decisions/` (ADR format)|
| Folder Structure | `CONTRIBUTING.md` |
| Known Issues | Open GitHub Issues with 'bug' label |
