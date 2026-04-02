# User Roles

> **Document status:** Living document - update when role permissions change.
> **Owner:** IT Coordinator
> **Last Reviewed:** April 2026

---

## 2.1 Role OverView

The system enforces four distinct roles. Roles are assigned at the database level and embedded in the JWT toekn at login. Every privileged operation is validated server-side - the frontend role state is for UI rendering only and is never trusted for access decisions.

| Role | Spring Security value | Typical holder | Assigned by |
|---|---|---|---|
| `ADMIN` | `ADMIN` | ChairPerson| IT Coordinator, manually |
| `MODERATOR` | `MODERATOR` | Active Board member | IT Coordinator, manually |
| `USER` (Member) | `USER` + `member_status: MEMBER` | Full LC member | IT Coordinator, manually |
| `USER` (Junior) | `USER` + `member_status: JUNIOR` | New Member | IT Coordinator, manually |

> **Board cycle note:** The `ADMIN` role rotates yearly with the board. At each cycle handover, the outgoing admin promotes the incoming admin promotes the incoming IT Coordinator and demotes themselves. This must be done before the outgoing admin's account is deactivated. There is currently no automated handover workflow - this is a manual operation.

---

## 2.2 Role Definitions

### ADMIN

THE `ADMIN` role is the highest privilege level in the system. It is held by the ChairPerson for the current board cycle.

**Capabilities:**
- All `MODERATOR` capabilities
- Create and delete usedr accounts
- Promote users to  `MODERATOR`
- Demote `MODERATOR` 
- Delete General Meetings
- Delete polls

**Constraints:** 
- There is no technical enforcement of a single admin - multiple admin accounts can exists simultaneousnly. This is intentional to allow handover overlap during bord transition.
- Admin actions are not currrently logged to an audit table. All changes are reflected in the database state only.

**Endpoints requiring ADMIN:**
- `DELETE api/users/{id}`
- `PATCH /api/users/{id}/promote-moderator`
- `PATCH /api/users/{id}/demote`
- `DELETE /api/general_meetings/{meetingId}`
- `DELETE /api/polls/{pollId}`

---

### MODERATOR

The `MODERATOR` role is held by active board members. It covers all operational actions required to run a General Meeting end-to-end.

**Capabilities:**
- Create General Meetings
- Create and delte attendance rounds within an active meeting
- check members in and out of attendance rounds
- Create polls with candidate options and electoral body count
- Close polls
- Create and delete Announcements
- Create events
- View all attendance records and poll results

**Constraints:**
- A moderator cannot promote or demote other users.
- A moderator cannot delte user accounts.
- Moderator access is validated on every privileged request via `ModeratoValidatorService` - passing a valid JWT is not sufficient if the token's user ID does not resolve to a `MODERATOR` or `ADMIN` role in the database.

**Endpoints requiring MODERATOR or higher:**
- `POST /api/general_meetings`
- `POST /api/attendance_rounds`
- `DELETE /api/attendance_rounds/{roundId}`
- `POST /api/attendance_check/check-in`
- `POST /api/attendance_check/check-out`
- `POST /api/polls`
- `PATCH /api/polls/{pollId}/close`
- `POST /api/announcements`
- `DELETE /api/announcements/{id}`

---

### USER - Full Member

A `USER` with `member_status:MEMBER` is a ful LC member with voting rights.

**Capabilities:**
- View the dashboard, events, announcement




