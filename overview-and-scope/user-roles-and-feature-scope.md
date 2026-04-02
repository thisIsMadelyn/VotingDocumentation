# User Roles

> **Document status:** Living document - update when role permissions change.
> **Owner:** IT Coordinator
> **Last Reviewed:** April 2026

---

## 2.1 Role OverView

The system enforces four distinct roles. Roles are assigned at the database level and embedded in the JWT toekn at login. Every privileged operation is validated server-side - the frontend role state is for UI rendering only and is never trusted for access decisions.

| Role | Spring Security value | Typical holder | Assigned by |
|---|---|---|---|
| `ADMIN` | `ADMIN` | ChairPerson| IT Coordinator, manually|




