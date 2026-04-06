# Use Cases

> **Document status:** Living document.
> **Owner:** IT Coordinator
> **Last revied:** April 2026

---

## Overview

Use cases describe the systeem from the user's perspective - what they are trying to accomplish and how the system responds. Each use case maps to one or more functional requirements and covers both the success path and most important failure conditions.

--- 

## UC-01 - Run a General Meeting End-to-End

**Actor:** Moderator
**Goal:** conduct Meeting with attendance tracking and voting.
**Related FRs:** FR-301, FR-401 to FR-409, FR-501 to FR-510

### Preconditions
- The moderator is logged in with a valid JWT.
- At least one full member (`member_status:MEMBER`) exists in the system.
- No other General Meeting is currently active.

### Main Flow

1. the moderator creates a new General Meeting via the backend (`POST /api/general_meetings`). The meeting is set as active.
2. The moderator navigates to the Attendance page and creates a new attendance round (Round 1) using the **+ New Round** button.
3. The moderator checks in each present member using the **check In** button on the round card.
4. When the meeting begins the voting agenda item, the moderator creates a new poll with the candidate names, majprity type and electoral body count.
5. Each eligible number navigates to the Polls page and casts their vote.
6. the modearator closes the poll when voting is complete.
7. If a second round is required, the moderator advances the poll to `SECOND_ROUND` and members vote again.
8. The moderator closes the meeting by seeting `is_active` to false.

### Failure Conditions

| Condition | System Response |
|---|---|
| A member tries to vote but is not checked in | `400 Bad Request` — "No attendance record found for user X in meeting Y" |
| A member tries to vote but their status is `JUNIOR` | `400 Bad Request` — "User X is not an active MEMBER" |
| A member tries to vote twice in the same round | `400 Bad Request` — "User has already voted in this round" |
| A second active meeting is created while one is already active | Current behaviour: both exist as active. DB-level enforcement is pending. |
| The moderator tries to create a round but no active meeting exists | `404 Not Found` from `GET /api/general_meetings/active` |

---

## UC-02 - Cast a Vote

**Actor:** Full Member (`member_status: MEMBER`)  
**Goal:** Cast a vote in an open poll during a General Meeting.  
**Related FRs:** FR-502, FR-503, FR-504, FR-505, FR-506

### Preconditions
- The user is logged in.
- An active General Meeting exists.
- The user has been into an attencence round for that meeting and has not checked out.
- A poll is open (`is_active: true`, `status: REQUIRES_NEXT_ROUND`).

###  Main Flow

1. The user navigates to the Polls page.
2. The user selects their preferred candidate option.
3. The system validates all eligibility conditions server-side.
4. The vote is recorded with the selected option, current round, and timestamp.
5. The user receives confirmation that their vote was cast.

### Failure Conditions

| Conditions | System Response |
|---|---|
| User not checked in | `400` — attendance check fails |
| User is a junior member | `400` — membership status check fails |
| User already voted this round | `400` — duplicate vote check fails |
| Selected option does not belong to the poll | `400` — option ownership check fails |
| Poll is closed | `400` — poll active check fails |

### Change Vote Flow

1. The user realises they want to change their vote before the poll closes.
2. The user submits a `PATCH /api/votes/{id}` request with the new option ID.
3. The system validates that the new option belongs to the same poll.
4. The vote record is updated with the new option and a new `voted_at` timestamp.

---

## UC-03 — Onboard a New Member

**Actor:** Admin  
**Goal:** Add a new member to the system with the correct role and membership status.  
**Related FRs:** FR-201, FR-207, FR-208

### Preconditions
- The admin is logged in.
- The new member's details are available (name, email, university).

### Main Flow

1. the admin creates a new user account via `POST /api/auth/register` with `member_status: JUNIOR` and `login_role: USER`.
2. The new member can now login and view the dashboard, announcements and events.
3. Once the member is formally approved by the board, the admin updates their `member_status` to `MEMBER` via `POST /api/users/{id}`
4. The member can now cast votes in General Meeting polls.

### Failure Conditions 

| Conditions | System Response |
|---|---|
| Username already exists | `400` — duplicate username rejection |
| Admin attempts to set role to `ADMIN` at registration | Not possible via the register endpoint — role promotion is a separate admin-only operation |

---

## UC-04 — Board Cycle Handover

**Actor:** Outgoing Admin, Incoming Admin  
**Goal:** Transfer admin access to the new IT Coordinator at the start of a board cycle.  
**Related FRs:** FR-203, FR-204

### Preconditions
- Both the outgoing and incoming IT Coordinator have active accounts.
- The outgoing admin's JWT is still valid.

### Main Flow
 
1. The outgoing admin promotes the incoming IT Coordinator to `MODERATOR` via `PATCH /api/users/{id}/promote-moderator`.
2. Since there is no `promote-admin` endpoint, the incoming coordinator's `roles_status` field must be updated directly in the database to `ADMIN` for the current release.
3. The outgoing admin demotes themselves to `USER` via `PATCH /api/users/{id}/demote`.

> **Note:** Step 2 requires direct database access. An admin promotion endpoint (`PATCH /api/users/{id}/promote-admin`) is a planned improvement to eliminate this dependency on manual DB access during handover.

### Failure Conditions

| Condition | System response |
|---|---|
| Outgoing admin's token has expired before handover completes | Must log in again. If the account has already been demoted, a superuser DB intervention is required. |

---

## UC-05 — Create and Manage an Announcement
 
**Actor:** Any authenticated user  
**Goal:** Post an organisation-wide announcement.  
**Related FRs:** FR-601, FR-602, FR-603, FR-604, FR-605

### Preconditions
- The user is logged in with any role.

### Main Flow
 
1. The user navigates to the Announcements page.
2. The user fills in the title, content, priority, and optional expiry date.
3. The system creates the announcement and automatically assigns the creating user as the author.
4. The announcement appears in the list sorted by creation date, with the author's name displayed.

### Failure Conditions

| Condition | System response |
|---|---|
| Title or content is missing | `400 Bad Request` |
| Priority is not a valid enum value | `400 Bad Request` |

### Deletion Flow
 
1. A moderator navigates to the Announcements page.
2. The moderator selects an announcement to delete.
3. The system permanently removes the announcement. This action is not reversible.

---

## UC-06 — Check a Member In and Out of an Attendance Round
 
**Actor:** Moderator  
**Goal:** Record a member's attendance for a specific round in an active meeting.  
**Related FRs:** FR-401, FR-402, FR-403, FR-404, FR-405

### Preconditions
- An active General Meeting exists.
- At least one attendance round has been created for that meeting.
- The moderator is logged in.

### Main Flow — Check In

1. The moderator opens the Attendance page and navigates to the Active Meeting tab.
2. The moderator expands the target round card.
3. All members are listed. Members not yet checked in show status **Absent**.
4. The moderator clicks **Check In** next to a member's name.
5. The system records an attendance record with `check_in_time` set to the current timestamp and `round_id` referencing the current round.
6. The member's status updates to **Present**.

### Main Flow — Check Out
 
1. The moderator clicks **Check Out** next to a checked-in member.
2. The system updates the attendance record with `check_out_time` set to the current timestamp.
3. The member's status updates to **Checked Out**.

### Failure Conditions

| Conditions | System Response |
|---|---|
| Member already checked in to this round | `409 Conflict` — "Attendance already recorded for this user in this round" |
| Member has already checked out | `409 Conflict` — "User has already checked out of this meeting" |
| No active meeting | Frontend shows empty state — "No active meeting right now" |

---

## UC-07 — Promote a Member to Moderator
 
**Actor:** Admin  
**Goal:** Grant board-level access to a member who has joined the board.  
**Related FRs:** FR-203

### Preconditions
- The admin is logged in.
- The target user has an active account.

### Main Flow
 
1. The admin calls `PATCH /api/users/{userId}/promote-moderator?adminId={adminId}`.
2. The system validates that `adminId` resolves to an `ADMIN` account.
3. The target user's `roles_status` is updated to `MODERATOR` in the database.
4. The change takes effect on the user's next login — their current token retains the old role until expiry.

### Failure Conditions

| Conditions | System Response |
|---|---|
| `adminId` does not resolve to an admin account | `403 Forbidden` |
| Target user does not exist | `404 Not Found` |

> **Important:** The promoted user must log out and log back in to receive a token with the updated role. Communicating this to the user is the admin's responsibility — the system does not force a session refresh.

