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
