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
- View the dashboard, events, announcement and member directory
- Cast a vote in an active poll, subject to all of the following conditions being true simultaneously:
   - The poll is active (`is_active: true`)
   - The poll status is `REQUIRES_NEXT_ROUND`
   - The user has an attendance record for the meeting associated with the poll
   - The user has not checked out of that meeting
   - the user has not already voted in the current round
- Change their vote within the same open round
- View their own profile

**Constraints:**
- Cannot submit reports to board members
- Vote eligibility is enforced at the service layer isn `VotingValidationService`. A user who meets all conditions in the UI but fails any server-side check will recieve a `400 Bad Request` with a descriptive error message.
- **Known Limitation:** The attendance check in `validateVoteEligibility` queries the first attendance record for the user in the meeting - it the meeting - it does not filter by round. In meeting with multiple attendance rounds, a user checked in to Round 1 and checked out before Round 2 may still pass the attendance gate. This is tracked as a known issue and must be resolved before multi-roun voting is used in a live General Meeting. See Know Issues.

---

### USER - Junior Member

A `USER` with `member_status: JUNIOR` is a new member who has not yet been granted full membership.

**Capabilities:**
- View the dashboard, events and announcements
- View the member directory

**constraints:**
- Submit reports to board members
- Cannot cast votes. The `member_status` check in `VotingValidationService` rejects any vote attempt from a non-`MEMBER` account with a `400` error.
- Cannot access attendance management or poll management pages - these are gated in the frontend by moderrator role check in the backend by endpoint-level validation.

---

## 2.3 Role Escalation & Demotion

Role changes are performed by the developer with direct database queries or an `ADMIN` via the following endpoints:

```
PATCH /api/users/{userId}/promote-moderator?adminID={adminId}
PATCH /api/users/{userId}/demote?adminID={adminId}
```

Both endpoints validate that `adminId` resolves to an active `ADMIN` account before applying the change. The `userId` is updated in the database immediately - the affected user's current JWT remains valid until expiry. **There is no token invalidation on role chaange.** If a moderator is demoted, their existing token will continue to pass moderator checks until it expires. For sensitive demotions, therecommended mitigation is to also delete and recreate the uesr account, or to wait for the natural toekn expity window.

This is a known architectural limitation to address before the system handles adversarial scenarios.

---

---

# Feature Scope Map

> **Legend:**
> Complete and working in production
> Complete with known limitations - see notes
> In progress
> Not started

---

## 3.1 Authentivation & Session Management

| Feature | Status | Notes |
|---|---|---|
| JWT login | ✅ | Token issued at `/api/auth/login`. contains username, role, userId. |
| JWT validation on protected routes | ✅ | Spring Security filter chain validates token on every request. |
| Role embedding in token | ✅ | Role resolved from DB a login|





