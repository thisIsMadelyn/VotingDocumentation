# API Reference

> **Document status:** Living document - update when endpoints are added or changes.
> **Owner:** IT Coordinator
> **Github mapping:** `architecture/api-reference.md`
> **Last reviewed:** April 2026
> **Interactive docs:** Available at `http://localhost:8080/swagger-ui.html` when the api is running.

---

## Overview

The backend exposes a RESTful JSON API under the `/api` base path. All endpoints except authentication require a valid JWT token in the `Authorization` header:

```
Authorization: Bearer <token>
```

The token is obtained from `POST /pi/auth/login` and must be included in every subsequent request. The Axions client in `src/services/Axiosclient.js` attaches this header auttomatically via a request interceptor.

**Base URL:**
- Development: `http://localhost:8080/api`
- Production: To be configured — depends on reverse proxy setup on the university VM.

---

## Role Legend
 
| Symbol | Meaning |
|---|---|
| 🔓 | Public — no authentication required |
| 🔑 | Any authenticated user |
| 🛡 | Moderator or Admin only |
| 👑 | Admin only |

---

## Endpoint Index

### Authentication — `/api/auth`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/auth/register` | 🔓 | Register a new user account |
| `POST` | `/api/auth/login` | 🔓 | Log in and receive a JWT token |
 
---
 
### Users — `/api/users`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/users` | 🔑 | Get all users |
| `GET` | `/api/users/user/username/{username}` | 🔑 | Get a user by username |
| `POST` | `/api/users` | 👑 | Create a new user |
| `PATCH` | `/api/users/{id}` | 👑 | Partially update a user |
| `DELETE` | `/api/users/{id}` | 👑 | Delete a user by ID |
| `DELETE` | `/api/users/username?username=` | 👑 | Delete a user by username |
| `PATCH` | `/api/users/{userId}/promote-moderator` | 👑 | Promote a user to Moderator |
| `PATCH` | `/api/users/{userId}/demote` | 👑 | Demote a Moderator to User |
 
---
 
### General Meetings — `/api/general_meetings`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/general_meetings` | 🔑 | Get all meetings |
| `GET` | `/api/general_meetings/active` | 🔑 | Get the currently active meeting |
| `POST` | `/api/general_meetings` | 🛡 | Create a new meeting |
| `DELETE` | `/api/general_meetings/{meetingId}` | 👑 | Delete a meeting |
 
---
 
### Attendance Rounds — `/api/attendance_rounds`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/attendance_rounds/meeting/{meetingId}` | 🛡 | Get all rounds for a meeting |
| `POST` | `/api/attendance_rounds?meetingId=` | 🛡 | Create a new round for a meeting |
| `DELETE` | `/api/attendance_rounds/{roundId}` | 🛡 | Delete a round |
 
---
 
### Attendance Checks — `/api/attendance_check`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/attendance_check` | 🛡 | Get all attendance records |
| `GET` | `/api/attendance_check/by-meeting/{meetingId}` | 🛡 | Get attendance records for a meeting |
| `GET` | `/api/attendance_check/by-round/{roundId}` | 🛡 | Get attendance records for a round |
| `GET` | `/api/attendance_check/meeting/{meetingId}` | 🛡 | Get attendance by meeting (requires `moderatorId` param) |
| `POST` | `/api/attendance_check/check-in` | 🛡 | Check a user into a round |
| `POST` | `/api/attendance_check/check-out` | 🛡 | Check a user out of a meeting |
| `DELETE` | `/api/attendance_check/{id}` | 🛡 | Delete an attendance record |
| `PATCH` | `/api/attendance_check/{id}` | 🛡 | Partially update an attendance record |
 
**Check-in request body:**
```json
{
  "meetingId": 1,
  "userId": 3,
  "roundId": 2,
  "method": "MANUAL"
}
```
 
**Check-out request params:** `?meetingId=1&userId=3`
 
---
 
### Polls — `/api/polls`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/polls` | 🔑 | Get all polls |
| `GET` | `/api/polls/search/title?title=` | 🔑 | Find a poll by title |
| `GET` | `/api/polls/check-moderator/{userId}` | 🔑 | Check if a user is a moderator |
| `GET` | `/api/polls/check-admin/{userId}` | 🔑 | Check if a user is an admin |
| `GET` | `/api/polls/user-role/{userId}` | 🔑 | Get a user's role |
| `POST` | `/api/polls?moderatorId=` | 🛡 | Create a new poll |
| `PATCH` | `/api/polls/{pollId}/close?moderatorId=` | 🛡 | Close a poll |
| `DELETE` | `/api/polls/{pollId}?adminId=` | 👑 | Delete a poll |
 
**Create poll request body:**
```json
{
  "meetingId": 1,
  "title": "Election of IT Coordinator",
  "description": "Annual election for the IT Coordinator position",
  "majorityType": "ABSOLUTE",
  "electoralBodyCount": 12,
  "candidateNames": ["Alice Papadopoulou", "Bob Nikolaou"]
}
```
 
---
 
### Votes — `/api/votes`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/votes` | 🛡 | Get all votes |
| `GET` | `/api/votes/search/poll?pollId=` | 🔑 | Get votes for a poll |
| `GET` | `/api/votes/search/user?userId=` | 🔑 | Get a user's vote |
| `POST` | `/api/votes` | 🔑 | Cast a vote |
| `PATCH` | `/api/votes/{id}` | 🔑 | Change a vote |
| `DELETE` | `/api/votes/{id}` | 👑 | Delete a vote |
 
**Cast vote request body:**
```json
{
  "userId": 3,
  "pollId": 1,
  "optionId": 2,
  "voteOption": "FOR"
}
```
 
**Vote eligibility — all of the following must be true:**
- Poll is active (`is_active: true`)
- Poll status is `REQUIRES_NEXT_ROUND`
- User has an attendance record for the meeting (not checked out)
- User `member_status` is `MEMBER`
- User has not already voted in the current round
 
---
 
### Announcements — `/api/announcements`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/announcements` | 🔑 | Get all announcements, newest first |
| `POST` | `/api/announcements` | 🔑 | Create an announcement |
| `DELETE` | `/api/announcements/{id}` | 🛡 | Delete an announcement |
 
**Create announcement request body:**
```json
{
  "title": "Scheduled Maintenance",
  "content": "The dashboard will be offline for 10 minutes at midnight.",
  "priority": "LOW"
}
```
 
> Author is automatically assigned from the JWT token — do not include it in the request body.
 
---
 
### Events — `/api/events`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/events` | 🔑 | Get all events |
| `POST` | `/api/events` | 🔑 | Create an event |
| `DELETE` | `/api/events/{id}` | 🛡 | Delete an event |
 
---
 
### Reports — `/api/reports`
 
| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/reports` | 🔑 | Get all reports |
| `GET` | `/api/reports/inbox/{userId}` | 🔑 | Get reports received by a user |
| `GET` | `/api/reports/sent/{userId}` | 🔑 | Get reports sent by a user |
| `POST` | `/api/reports` | 🔑 | Submit a report |
| `DELETE` | `/api/reports/{id}` | 🔑 | Delete a report |
 
---
 
## Common Error Responses
 
All error responses follow this shape:
 
```json
{
  "status": 400,
  "message": "Descriptive error message"
}
```
 
> **Note:** A global `@ControllerAdvice` exception handler is not yet implemented. Some endpoints return plain text error messages rather than structured JSON. This must be standardised before production.
 
| Status | Meaning |
|---|---|
| `400 Bad Request` | Validation failure or business rule violation |
| `401 Unauthorized` | Missing or invalid JWT token |
| `403 Forbidden` | Valid token but insufficient role |
| `404 Not Found` | Requested resource does not exist |
| `409 Conflict` | Duplicate resource (already checked in, already voted) |
| `500 Internal Server Error` | Unhandled exception — must not occur in production |
 
---
 
## Interactive API Documentation
 
The full API is documented with Swagger annotations and is available interactively at:
 
```
http://localhost:8080/swagger-ui.html
```
 
The Swagger UI allows you to inspect every endpoint's full request/response schema, try requests directly against the running backend, and view all `@ApiResponse` annotations. For detailed field-level documentation, always refer to Swagger as the authoritative source — it is generated directly from the codebase and stays in sync automatically.
 
---
 
## Planned Endpoints
 
The following endpoints are needed but not yet implemented:
 
| Method | Path | Purpose |
|---|---|---|
| `PATCH` | `/api/users/{id}/promote-admin` | Promote a user to Admin without direct DB access |
| `POST` | `/api/polls/{pollId}/next-round` | Advance a poll to the next election round |
| `GET` | `/api/polls/{pollId}/results` | Get vote counts and winner for a closed poll |
| `POST` | `/api/auth/refresh` | Refresh a JWT token without re-authenticating |
