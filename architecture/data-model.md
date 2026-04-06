# Data Model

> **Document status:** Living document — update after every Liquibase migration.  
> **Owner:** IT Coordinator  
> **Last reviewed:** April 2026  
> **GitHub mapping:** `db/changelog/` (Liquibase migrations)  
> **ERD:** See attached DataGrip export (`architecture/vms-erd.png`)

--- 

## Overview

The database is MySQL 8, managed exclusively through Liquibase versioned migrations. The schema consists of 12 application tables and 2 Liquibase system tables (`databasechangelog`, `databasechangeloglock`). No application code writes directly to the Liquibase system tables.

All schema changes must be made through a new Liquibase changeset. Manual `ALTER TABLE` statements against the production database are prohibited — see NFR-201.

---

## Enity Summary

| Table | Purpose | Key relationships |
|---|---|---|
| `users` | All LC member accounts | Central hub — referenced by most tables |
| `general_meetings` | General Meeting sessions | Parent of `attendance_round`, `attendance_check`, `polls` |
| `attendance_round` | A single attendance round within a meeting | Child of `general_meetings`, parent of `attendance_check` |
| `attendance_check` | Individual member check-in/out record | References `users`, `general_meetings`, `attendance_round` |
| `polls` | A voting poll tied to a meeting | Child of `general_meetings`, parent of `poll_options`, `votes` |
| `poll_options` | Candidate options for a poll | Child of `polls`, referenced by `votes` |
| `votes` | Individual vote cast by a member | References `users`, `polls`, `poll_options` |
| `committees` | LC committees | Parent of `committee_membership` |
| `committee_membership` | Member-to-committee assignment | References `users`, `committees` |
| `announcements` | Organisation-wide announcements | References `users` (author) |
| `events` | LC events | Standalone — no foreign keys |
| `reports` | Member-to-member reports | References `users` twice (sender, receiver) |

---

## Entity Definitions 

---

### `users`
 
The central identity table. Every authenticated action in the system traces back to a user record.
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `int` | No | Primary key, auto-increment |
| `username` | `varchar(255)` | No | Unique. Used as the JWT subject claim. |
| `password` | `varchar(255)` | No | BCrypt hash. Never returned in API responses. |
| `first_name` | `varchar(255)` | Yes | |
| `last_name` | `varchar(255)` | Yes | |
| `user_email` | `varchar(255)` | Yes | |
| `user_phone_num` | `varchar(255)` | Yes | |
| `discord_tag` | `varchar(255)` | Yes | |
| `date_of_birth` | `int` | Yes | Stored as integer — review before production. |
| `university` | `enum('auth','thu','bamaet')` | Yes | LC member's university affiliation. |
| `member_status` | `varchar(255)` | Yes | `MEMBER`, `JUNIOR`, or `ALUMNI`. Controls voting eligibility. |
| `roles_status` | `int` | Yes | Maps to `LoginRole` enum. Controls API access level. |
| `login_property` | `enum('admin','moderator','user')` | Yes | Redundant with `roles_status` — review for consolidation. |
 
**Design notes:**
- `member_status` and `roles_status` serve different purposes. A user can be a `MODERATOR` (board role) but have `member_status: JUNIOR` if they joined the board before completing membership. Both fields are checked independently.
- `login_property` appears to duplicate `roles_status`. This should be reviewed and consolidated before production to avoid inconsistency.
- `date_of_birth` is typed as `int` in the DB but mapped as `Date` in the Java entity — verify the column type is correct in the next migration.
 
---

### `general_meetings`

Represents a single General Meeting sesson of the LC.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `int` | No | Primary key, auto-increment |
| `meeting_date` | `datetime(6)` | No | Date and time of the meeting |
| `quorum_required` | `varchar(255)` | No | Minimum attendance for decisions to be binding. Advisory only — not programmatically enforced. |
| `is_active` | `bit(1)` | No | Flags the currently active meeting. Default `0`. At most one row should have `is_active = 1` at any time. |

**Design notes:**
- `is_active` is enforced at the application level only. A DB-level unique partial index on `is_active = 1` is a planned improvement to guarantee the single-active-meeting invariant.
- `quorum_required` is stored as `varchar` to allow values like `"60%"` or `"absolute majority"`. If programmatic enforcement is added, this should be migrated to a numeric type.

---

### `attendance_round`
 
Represents one round of attendance within a General Meeting. A meeting can have multiple rounds (e.g. start of meeting, post-break recount).
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `int` | No | Primary key, auto-increment |
| `meeting_id` | `int` | No | FK → `general_meetings.id` |
| `round_number` | `int` | No | Sequential number within the meeting. Auto-assigned by the service layer. |
| `created_at` | `datetime` | No | Timestamp when the round was opened by the moderator. |
 
---
 
### `attendance_check`
 
Records an individual member's check-in and check-out for a specific attendance round.
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `attendance_id` | `int` | No | Primary key, auto-increment |
| `meeting_id` | `int` | No | FK → `general_meetings.id` |
| `user_id` | `int` | No | FK → `users.id` |
| `round_id` | `int` | Yes | FK → `attendance_round.id`. Nullable for records created before rounds were introduced. |
| `check_in_time` | `datetime(6)` | No | Timestamp of check-in. |
| `check_out_time` | `datetime` | Yes | Timestamp of check-out. Null means the member is currently present. |
| `attendance_method` | `varchar(255)` | No | `MANUAL`, `REMOTE`, or `PHYSICAL`. Maps to `AttendanceMethod` enum. |
 
**Design notes:**
- `round_id` is nullable to preserve backwards compatibility with records created before multi-round attendance was introduced. All new records must have a `round_id`.
- A member's attendance status is derived from this table: no record = absent, record with null `check_out_time` = present, record with non-null `check_out_time` = checked out.
- ⚠️ **Known limitation:** `VotingValidationService.validateVoteEligibility` queries `findByMeetingAndUser` which returns the first matching record regardless of round. A member checked out of Round 1 but not checked into Round 2 may still pass the attendance gate. This must be fixed before multi-round voting is used in production.
 
---
 
### `polls`
 
A voting poll created within a General Meeting.
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `int` | No | Primary key, auto-increment |
| `meeting_id` | `int` | No | FK → `general_meetings.id` |
| `title` | `varchar(255)` | No | Must be unique across all polls. |
| `description` | `varchar(255)` | No | |
| `is_active` | `bit(1)` | No | Whether the poll is currently accepting votes. |
| `current_round` | `enum('first_round','second_round','third_round','tie_breaker')` | No | Current election round. |
| `majority_type` | `enum('absolute','relative','two_thirds')` | No | Determines how the winner is calculated. |
| `poll_status` | `enum('requires_next_round','tie_requires_new_board_vote','winner_declared')` | No | Current lifecycle state of the poll. |
| `electoral_body_count` | `int` | No | Total number of eligible voters at poll creation time. Used as the denominator for majority calculations. |
| `created_at` | `datetime(6)` | No | |
 
**Design notes:**
- `electoral_body_count` is set at poll creation and does not change if members check in or out after the poll opens. This is intentional — the electoral body is fixed at the moment voting begins.
- The `third_round` value in `current_round` enum is defined in the DB but not in the `ElectionRound` Java enum. Verify alignment before production.
 
---
 
### `poll_options`
 
The candidate options available within a poll. Each option is a candidate name or choice.
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `int` | No | Primary key, auto-increment |
| `poll_id` | `int` | No | FK → `polls.id`. Cascade delete — options are deleted when the poll is deleted. |
| `option_text` | `varchar(255)` | No | Candidate name or choice label. |
 
---
 
### `votes`
 
Records an individual vote cast by a member in a specific poll round.
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `int` | No | Primary key, auto-increment |
| `user_id` | `int` | No | FK → `users.id` |
| `poll` | `int` | No | FK → `polls.id`. Column named `poll`, not `poll_id`. |
| `option_id` | `int` | No | FK → `poll_options.id` |
| `round` | `enum('first_round','second_round','third_round','tie_breaker')` | No | Round in which the vote was cast. |
| `vote_type` | `enum('against','blank','for')` | No | Maps to `VoteOption` enum. |
| `voted_at` | `datetime(6)` | No | Timestamp of the vote. |
 
**Constraints:**
- `UNIQUE(user_id, poll, round)` — enforced at the database level. A user cannot vote more than once per round in the same poll. This constraint is in addition to the application-layer check in `VotingValidationService`.
 
**Design notes:**
- The FK column is named `poll` rather than `poll_id` — this is inconsistent with the naming convention used elsewhere. Note this when writing queries or adding new columns.
 
---
 
### `committees`
 
Represents an LC committee (e.g. IT, Events, PR).
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `committee_id` | `int` | No | Primary key, auto-increment |
| `committee_name` | `varchar(255)` | No | Unique. |
| `vote_weight` | `int` | No | Default `1`. Intended for weighted voting scenarios. Not currently used in vote calculations. |
| `mandate` | `varchar(255)` | No | Short description of the committee's responsibilities. |
 
---
 
### `committee_membership`
 
Tracks a member's assignment to a committee, including their role and tenure dates.
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `membership_id` | `int` | No | Primary key, auto-increment |
| `user_id` | `int` | No | FK → `users.id` |
| `committee_id` | `int` | No | FK → `committees.committee_id` |
| `role_title` | `varchar(50)` | No | e.g. `"Head"`, `"Member"`, `"Deputy"` |
| `start_date` | `date` | No | When the assignment began. |
| `end_date` | `date` | Yes | When the assignment ended. Null means currently active. |
 
---
 
### `announcements`
 
Organisation-wide announcements visible to all authenticated users.
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `int` | No | Primary key, auto-increment |
| `title` | `varchar(255)` | No | |
| `content` | `varchar(255)` | No | |
| `priority` | `enum('high','low','medium')` | No | Maps to `AnnouncementPriority` enum. |
| `created_at` | `datetime` | Yes | Auto-set by Spring Data auditing. |
| `expires_at` | `datetime` | Yes | Optional expiry. Not currently enforced in queries — expired announcements still appear in API responses. |
| `user_id` | `int` | Yes | FK → `users.id`. Author of the announcement. Added in migration `005`. |
| `created_by` | `int` | Yes | FK → `users.id`. Added in migration `005`. Review for consolidation with `user_id` — both reference the author. |
 
**Design notes:**
- ⚠️ Both `user_id` and `created_by` appear in the ERD and both reference `users.id`. This is likely a duplication — one column should be removed. Verify which is used by the Java entity (`createdBy`) and remove the other via a migration before production.
- `expires_at` enforcement must be added to `AnnouncementRepository.findAllByOrderByCreatedAtDesc()` before production.
 
---
 
### `events`
 
LC events such as workshops, socials, or external activities.
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `int` | No | Primary key, auto-increment |
| `title` | `varchar(255)` | No | |
| `description` | `varchar(255)` | No | |
| `start_time` | `datetime(6)` | No | |
| `end_time` | `datetime(6)` | No | |
| `location` | `varchar(255)` | Yes | |
| `created_at` | `datetime(6)` | Yes | Auto-set by Spring Data auditing. |
 
**Design notes:**
- `events` has no foreign keys. It is intentionally standalone in the current release.
 
---
 
### `reports`
 
Member-to-member reports, used for formal communication directed at the board.
 
| Column | Type | Nullable | Notes |
|---|---|---|---|
| `report_id` | `int` | No | Primary key, auto-increment |
| `sender_id` | `int` | No | FK → `users.id`. The user submitting the report. |
| `receiver_id` | `int` | No | FK → `users.id`. The user receiving the report. |
| `title` | `varchar(255)` | No | |
| `content` | `varchar(300)` | No | Slightly longer than other text fields — intentional. |
| `created_at` | `datetime(6)` | Yes | Set by `@PrePersist` in the entity. |
 
---
 
## Schema Issues to Resolve Before Production
 
The following inconsistencies were identified during documentation and must be addressed via Liquibase migrations before deployment:
 
| Issue | Table | Action |
|---|---|---|
| `date_of_birth` typed as `int` in DB, `Date` in Java | `users` | Verify and correct column type |
| `login_property` duplicates `roles_status` | `users` | Review and consolidate — remove one |
| `poll` column named inconsistently (should be `poll_id`) | `votes` | Rename via migration — update all queries |
| `third_round` in DB enum not in Java `ElectionRound` enum | `polls`, `votes` | Align DB enum with Java enum |
| `user_id` and `created_by` both reference the author | `announcements` | Remove the unused column via migration |
| `expires_at` not enforced in queries | `announcements` | Add filter to repository query |
| `is_active = 1` not DB-enforced as unique | `general_meetings` | Add unique partial index |
| `round_id` nullable on new records | `attendance_check` | Consider making non-nullable after data backfill |
















