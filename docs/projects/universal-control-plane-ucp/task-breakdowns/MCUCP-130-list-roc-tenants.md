# Task Breakdown — MCUCP-130: US 1.2 — List my ROC Tenants

**Story:** [MCUCP-130](https://jira.rakuten-it.com/jira/browse/MCUCP-130)
**Parent task:** MCUCP-263

**Business scope:** Any authenticated UCP user can see which of their ROC tenants are subscribed to UCP and what their UCP role is in each. Users with no UCP role on a subscribed tenant see a clear message directing them to contact their tenant-admin. Tenants not subscribed to UCP are not shown.

---

## API Endpoint

### `GET /api/v1/me/tenants`
Return the authenticated user's ROC tenants filtered to UCP-subscribed tenants only. No permission gate — available to any authenticated user.

**Response shape per tenant:**

| Field | Source | Notes |
|---|---|---|
| `name` | JWT `groups` claim (tenant slug) | |
| `rns` | JWT `groups` claim | `rns:roc:iam::{tenant-slug}` |
| `ocRole` | JWT `groups` claim (`iam` service entry) | `"Tenant Admin"` or `"Tenant Member"` |
| `ucpRole` | See open question below | `"tenant-admin"`, `"developer"`, `"viewer"`, or `null` |
| `admins` | `oc_tenant_members` table (OC admins for this tenant) | Populated only when `ucpRole` is null, so user knows who to contact |

Filter: only tenants present in `ucp_registered_tenants` are included in the response. Unsubscribed ROC tenants are excluded.

For tenants where `ucpRole` is null, include the `admins` array (name + email of UCP `tenant-admin` users for that tenant) so the user knows who to request access from.

### User's tenant list from JWT `groups` claim
Parse `rns:roc:iam::{tenant-slug}:roles:{admin|member}` entries from the JWT `groups` claim to derive the user's ROC tenant memberships and ROC tenant role. This avoids a Horizon `GET /v0/members/{email}/tenants` call for the logged-in user's own data.

> **Open question:** whether OC Tenant Members have a `rns:roc:iam::{tenant}:roles:member` entry in their JWT `groups` claim is unconfirmed (only tested with a Tenant Admin account). Needs verification with a non-admin test account. This affects how membership is detected — if the `iam` entry is absent for members, tenant discovery requires a different signal.

### UCP role lookup
Resolve the user's UCP role per tenant. The source depends on the role resolution approach chosen in **MCUCP-138**:
- **JWT-based (PRD approach):** parse `rns:roc:ucp::{tenant-slug}:roles:{role}` from the JWT `groups` claim
- **DB-backed (current implementation):** query `tenant_role_assignments` for the user + tenant

Implement whichever approach is decided in MCUCP-138. This endpoint must be consistent with how `RequirePermission` resolves roles everywhere else.

---

## CLI Command

### `ucp tenants list` command
Call `GET /api/v1/me/tenants` and render the response as a table:

```
NAME             ROC ROLE       UCP ROLE
coupon-team      Tenant Admin   tenant-admin
points-team      Tenant Member  developer
loyalty-shared   Tenant Member  - (contact your tenant-admin to get a UCP role assigned in ROC Portal)
```

Column widths should pad to align. `UCP ROLE: -` is shown when `ucpRole` is null in the response, with the inline hint.

**Empty state** (no UCP-subscribed tenants):
```
No UCP-subscribed tenants found.
Ask your ROC Tenant Admin to subscribe your tenant to UCP in ROC Portal → Tenant Management → Subscriptions.
```

No audit event required — this is a read-only operation.

---

## Dependencies
- Requires `ucp_registered_tenants` table (populated by tenant registration flow — part of tenant onboarding work)
- UCP role resolution strategy must be decided as part of **MCUCP-138** before this endpoint is finalized
- JWT `groups` claim parsing helper is shared with MCUCP-129 (authentication) and MCUCP-138 (RBAC)

## Open Questions
1. **JWT `groups` format for Tenant Members** — whether OC Tenant Members have a `rns:roc:iam::{tenant}:roles:member` entry in their JWT. Needs confirmation with a non-admin account before tenant discovery logic is finalized.
2. **UCP role source** — must align with the decision made in MCUCP-138 (JWT-based vs DB-backed). This endpoint cannot be finalized until that decision is made.
