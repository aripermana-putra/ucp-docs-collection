# Task Breakdown — MCUCP-138: US 1.5 — UCP RBAC Model

**Story:** [MCUCP-138](https://jira.rakuten-it.com/jira/browse/MCUCP-138)
**Parent task:** MCUCP-263

**Business scope:** All UCP API endpoints enforce a 3-role permission model (`tenant-admin`, `developer`, `viewer`) per tenant. Roles are assigned by Tenant Admins via ROC Portal. UCP enforces permissions on every request and logs authorization failures. Users can inspect their own role and available operations on any of their tenants.

---

## Role Model

### `viewer` role addition
Add `viewer` as a new role to the permission model. `viewer` has read-only access — it can view inventory, provisioning history, workflow status, quota usage, audit logs, and registered GCP projects, but cannot provision, approve, manage, or configure anything.

Update the `RolePermissions` map:
```
viewer:       PermRead
developer:    PermRead | PermProvision
tenant-admin: PermRead | PermProvision | PermApprove | PermManage
```

Update any `role IN (...)` DB constraints to include `viewer`. Confirm no existing code paths assume only two tenant-level roles.

### Permission matrix verification
Audit every registered route against the full permission matrix and ensure each route is wired to the correct `RequirePermission` gate:

| Operation | Required permission |
|---|---|
| List/get any resource, history, quota, audit log | `PermRead` |
| Provision or delete resources | `PermProvision` |
| Cancel in-progress workflow | `PermProvision` |
| Approve or reject provisioning workflows | `PermApprove` |
| Register / update GCP projects | `PermManage` |
| Update quota limits | `PermManage` |
| Configure approval policies | `PermManage` |
| Configure drift response policy | `PermManage` |

`viewer` passes all `PermRead` gates and is rejected (403) on all others.

---

## UCP Role Resolution

### Role source: JWT vs DB — architectural decision
The PRD specifies that UCP roles are read from the JWT `groups` claim on every request, with UCP registered as a service in ROC Core Data (`rns:roc:ucp::{tenant-slug}:roles:{role}`). The current implementation reads from the `tenant_role_assignments` DB table populated by a login-triggered sync.

**This is the most critical open question for this story.** Both approaches are valid but mutually exclusive:

- **JWT-based (PRD direction):** parse `rns:roc:ucp::` entries from the JWT `groups` claim on every request. No DB role lookup at request time. Role changes in ROC Portal take effect within one token TTL (≤8 hours). Requires UCP to be registered as a service in ROC Core Data with all three roles defined.
- **DB-backed (current implementation):** read from `tenant_role_assignments` at request time. Login-triggered sync keeps the table current. Role changes take effect at the user's next login. No dependency on ROC Core Data service registration.

> **Open question (must decide before this story starts):** Is UCP registered in ROC Core Data with `ucp` service roles (`tenant-admin`, `developer`, `viewer`) defined? If yes, the JWT-based approach is feasible and preferred (no sync logic needed). If not, the DB-backed approach remains and sync logic must be maintained. This decision drives the entire implementation of this story.

### JWT `groups` claim parser for UCP roles
_(Conditional on JWT-based approach being chosen)_

Parse `rns:roc:ucp::{tenant-slug}:roles:{role}` entries from the JWT `groups` claim. Map the parsed role string to the internal Role type. This parser is an extension of the same helper used for OC tenant/service role parsing — reuse the same parsing logic.

### `RequirePermission` middleware
Ensure the middleware correctly handles all three roles including `viewer`. `viewer` must resolve to a valid non-zero role (not `RoleUnknown`) and pass `PermRead` checks. Update role resolution to read from whichever source is decided (JWT or DB). The resolved role is injected into the request context for handler use.

---

## 403 Error Responses

### Standardize 403 response format
Ensure all 403 responses produced by `RequirePermission` use a consistent format:

When the user has no UCP role on the target tenant:
```json
{ "code": "FORBIDDEN", "message": "You do not have a UCP role on tenant 'coupon-team'. Contact your tenant-admin to get a role assigned in ROC Portal." }
```

When the user has a role but it is insufficient for the operation:
```json
{ "code": "FORBIDDEN", "message": "You do not have permission to perform this operation on tenant 'coupon-team'." }
```

Implement in the `RequirePermission` middleware layer — not per-handler. Audit existing handlers for any inline 403 responses that bypass the middleware format.

---

## User-Facing Role Visibility

### `ucp whoami` command
Without `--tenant`: return all tenants the user has a UCP role in, with the role per tenant.

With `--tenant <slug>`:
```
Tenant: coupon-team
UCP Role: developer
Permitted operations: provision resources, view inventory & cost, view provisioning history, view quota, view audit log, view registered GCP projects
```

Derive the permitted operations list from the `RolePermissions` map at runtime — do not hardcode it in the CLI output layer.

### `GET /api/v1/me/whoami` (or extend `/api/v1/me`)
Back the `ucp whoami` CLI command. Return the user's role per tenant and the derived permitted operations list. Accessible to any authenticated user regardless of role.

---

## Authorization Failure Audit Logging

### Authz failure logging in middleware
On every 403 response from `RequirePermission`, write an audit log entry:

| Field | Value |
|---|---|
| `user_id` | resolved UCP user ID |
| `action` | `authorization_failed` |
| `tenant` | target tenant RNS (if known from request context) |
| `required_role` | the permission that was required |
| `timestamp` | event time |

Implement in the `RequirePermission` middleware layer, not per-handler. Do not log 401 responses here — those are covered by the auth middleware (MCUCP-129).

---

## Dependencies
- MCUCP-130 (`ucp tenants list`) depends on the UCP role resolution approach finalized in this story
- `viewer` role must be defined in ROC Core Data before it can be assigned via ROC Portal (if JWT-based approach is adopted)
- `platform-admin` is out of scope for MVP tenant-level work — existing `platform-admin` rows in `tenant_role_assignments` (with `tenant_rns = '*'`) should be preserved as-is (manual assignment only, untouched by any sync)

## Open Questions
1. **JWT vs DB role resolution** — most critical decision; must be resolved before this story starts. Depends on whether UCP is registered in ROC Core Data with `ucp` service roles.
2. **`viewer` role in ROC Core Data** — requires coordination with the ROC team to define the role in Core Data before it can be assigned via ROC Portal. Clarify timeline dependency.
3. **Login-triggered sync (if DB-backed approach retained)** — role changes between logins are reflected only at next login. Is this staleness window acceptable for MVP, or is a background sync job required?
4. **Background sync** — if DB-backed: define the acceptable staleness window and whether a periodic sync is needed for MVP.
5. **`platform-admin` scope** — clarify whether the existing `platform-admin` role (cross-tenant, `tenant_rns = '*'`) is in or out of MVP scope. PRD-003 defers platform-level roles to Phase 2. The current implementation includes it — decide whether to keep or remove from active use.
