# Task Breakdown — MCUCP-130: US 1.2 — List my ROC Tenants

**Story:** [MCUCP-130](https://jira.rakuten-it.com/jira/browse/MCUCP-130)
**Parent task:** MCUCP-263

**Business scope:** Any authenticated UCP user can see which of their ROC tenants are subscribed to UCP and what their UCP role is in each. Users with no UCP role on a subscribed tenant see a clear message directing them to contact their tenant-admin. Tenants not subscribed to UCP are not shown.

---

## Subtask 1: `GET /api/v1/me/tenants` endpoint
**Components:** API Server, Platform DB
**Blocked by:** MCUCP-129 Subtask 1 (JWT validation), MCUCP-138 Subtask 1 (DB schema), MCUCP-138 Subtask 3 (role resolution middleware)
**Blocks:** Subtask 2

### API Contract

**`GET /api/v1/me/tenants`**

Returns the authenticated user's ROC tenants filtered to UCP-subscribed tenants only. No permission gate — available to any authenticated user regardless of UCP role.

| | |
|---|---|
| Auth required | Yes (Bearer token or session cookie) |
| Request body | None |

**Response `200 OK`:**
```json
{
  "items": [
    {
      "name": "coupon-team",
      "rns": "rns:roc:iam::coupon-team",
      "ocRole": "Tenant Admin",
      "ucpRole": "tenant-admin",
      "admins": null
    },
    {
      "name": "points-team",
      "rns": "rns:roc:iam::points-team",
      "ocRole": "Tenant Member",
      "ucpRole": "developer",
      "admins": null
    },
    {
      "name": "loyalty-shared",
      "rns": "rns:roc:iam::loyalty-shared",
      "ocRole": "Tenant Member",
      "ucpRole": null,
      "admins": [
        { "name": "Alice Smith", "email": "alice@rakuten.com" }
      ]
    }
  ]
}
```

**Field rules:**
- `items` is an empty array (not null) when the user has no UCP-subscribed tenants
- `ucpRole` is `null` when the tenant is UCP-subscribed but the user has no role assigned
- `admins` is populated (from `oc_tenant_members`) only when `ucpRole` is `null`, so the user knows who to contact
- Tenants not present in `ucp_registered_tenants` are excluded entirely

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Always (authenticated) | Response above; `items` may be empty |
| `401 Unauthorized` | Not authenticated | Standard `UNAUTHENTICATED` body |

**Implementation notes:**
- Parse `rns:roc:iam::{tenant-slug}:roles:{admin|member}` entries from the JWT `groups` claim to derive the user's ROC tenant memberships and ROC tenant role — no Horizon API call needed for the logged-in user's own data
- Resolve UCP role using whichever approach is decided in MCUCP-138 (JWT-based or DB-backed via `tenant_role_assignments`)

> **Open question:** whether OC Tenant Members have a `rns:roc:iam::{tenant}:roles:member` entry in their JWT is unconfirmed (tested with Tenant Admin only). Needs verification with a non-admin account before this endpoint is finalized. This affects how tenant membership is detected from the JWT.

---

## Subtask 2: `ucp tenants list` CLI command
**Components:** CLI
**Blocked by:** Subtask 1

### CLI Definition

```
ucp tenants list
```

No flags. Calls `GET /api/v1/me/tenants` and renders as a padded table.

| Outcome | Output |
|---|---|
| Tenants found | See table below |
| No UCP-subscribed tenants | See empty state below |
| Not authenticated | `Error: not authenticated. Run 'ucp auth login'.` |

**Tenants found:**
```
NAME             ROC ROLE       UCP ROLE
coupon-team      Tenant Admin   tenant-admin
points-team      Tenant Member  developer
loyalty-shared   Tenant Member  - (contact your tenant-admin to get a UCP role assigned in ROC Portal)
```

Column widths pad to the longest value in each column.

**Empty state:**
```
No UCP-subscribed tenants found.
Ask your ROC Tenant Admin to subscribe your tenant to UCP in ROC Portal → Tenant Management → Subscriptions.
```

No audit event required — read-only operation.

---

## Open Questions
1. **JWT `groups` format for Tenant Members** — whether OC Tenant Members have a `rns:roc:iam::{tenant}:roles:member` JWT entry. Must be confirmed with a non-admin test account before Subtask 1 is finalized.
2. **UCP role source** — must align with the decision made in MCUCP-138 (JWT-based vs DB-backed). Subtask 1 cannot be finalized until that decision is made.
