# Task Breakdown — MCUCP-130: US 1.2 — List my ROC Tenants

**Story:** [MCUCP-130](https://jira.rakuten-it.com/jira/browse/MCUCP-130)
**Parent task:** MCUCP-263

**Business scope:** Any authenticated UCP user can see which of their ROC tenants are subscribed to UCP and what their UCP role is in each. Users with no UCP role on a subscribed tenant see a clear message directing them to contact their tenant-admin. Tenants not subscribed to UCP are not shown.

**Codebase:** monorepo at `ucp-platform/`. Feature slice implementation in `api-server/internal/tenant/`. API contract defined in `api-server/api/openapi.yaml`.

---

## Subtask 1: `GET /api/v1/me/tenants` endpoint
**Components:** API Server, Platform DB
**Blocked by:** MCUCP-129 Subtask 1 (JWT middleware), MCUCP-138 Subtask 1 (DB schema), MCUCP-138 Subtask 3 (role resolution middleware)
**Blocks:** Subtask 2

### API Contract
Add to `api-server/api/openapi.yaml`:

```yaml
paths:
  /api/v1/me/tenants:
    get:
      operationId: listMyTenants
      summary: List UCP-subscribed tenants for the authenticated user
      tags: [tenants]
      security:
        - sessionCookie: []
        - bearerAuth: []
      responses:
        "200":
          description: UCP-subscribed tenants. Returns empty items array when none found.
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/TenantListResponse"
        "401":
          $ref: "#/components/responses/Unauthorized"

components:
  schemas:
    TenantListResponse:
      type: object
      required: [items]
      properties:
        items:
          type: array
          items:
            $ref: "#/components/schemas/TenantItem"

    TenantItem:
      type: object
      required: [name, rns, ocRole]
      properties:
        name:
          type: string
          example: coupon-team
        rns:
          type: string
          example: "rns:roc:iam::coupon-team"
        ocRole:
          type: string
          enum: [Tenant Admin, Tenant Member]
        ucpRole:
          type: string
          nullable: true
          enum: [tenant-admin, developer, viewer]
          description: null when tenant is UCP-subscribed but user has no role assigned
        admins:
          type: array
          nullable: true
          description: UCP tenant-admins to contact when ucpRole is null
          items:
            $ref: "#/components/schemas/AdminContact"

    AdminContact:
      type: object
      required: [name, email]
      properties:
        name:
          type: string
        email:
          type: string
          format: email
```

### Implementation
Implement in `api-server/internal/tenant/`:

- Parse `rns:roc:iam::{tenant-slug}:roles:{admin|member}` entries from the JWT `groups` claim (from `Principal` in context) to derive the user's ROC tenant memberships and ROC tenant role — no Horizon API call for the logged-in user's own data
- Filter to tenants present in `ucp_registered_tenants` — exclude unsubscribed ROC tenants
- Resolve UCP role using the approach decided in MCUCP-138 (JWT-based or DB-backed via `tenant_role_assignments`)
- When `ucpRole` is null: populate `admins` from `oc_tenant_members` (OC Tenant Admins for that tenant) so the user knows who to contact
- No permission gate — available to any authenticated user regardless of UCP role

> **Open question:** whether OC Tenant Members have a `rns:roc:iam::{tenant}:roles:member` entry in their JWT `groups` claim is unconfirmed (tested with Tenant Admin accounts only). Needs verification with a non-admin account before this handler is finalized — affects how membership is detected from the JWT.

---

## Subtask 2: `ucp tenants list` CLI command
**Components:** CLI
**Blocked by:** Subtask 1

### CLI Definition
Implement in `cli/`. Uses the generated client from `cli/gen/client.gen.go` to call `GET /api/v1/me/tenants`.

```
ucp tenants list
```

No flags.

| Outcome | Output |
|---|---|
| Tenants found | Table (see below) |
| No UCP-subscribed tenants | Empty state (see below) |
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
1. **JWT `groups` format for Tenant Members** — whether OC Tenant Members have a `rns:roc:iam::{tenant}:roles:member` entry in their JWT. Must be confirmed with a non-admin account before Subtask 1 is finalized.
2. **UCP role source** — must align with the decision made in MCUCP-138 (JWT-based vs DB-backed). Subtask 1 cannot be finalized until that decision is made.
