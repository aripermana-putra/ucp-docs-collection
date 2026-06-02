---
title: "RBAC"
space: UCP
parent_page_id: "../pocs.md"
---

# RBAC

UCP enforces access control at the API server layer using a
**5-role per-tenant model**. Each role defines the minimum set of operations
a user can perform within a tenant. Role assignments are stored in UCP's own
PostgreSQL database (`tenant_role_assignments` table) — independent of
Keycloak and Horizon.

---

## Approach Summary

Every authenticated request passes through a `RequireRole(minRole)` middleware
before reaching the handler. The middleware loads the caller's role assignments
from the DB in a single call, caches them in the request context, and enforces
the minimum role. Requests that do not meet the minimum required role are
rejected with HTTP 403.

`platform-admin` is a cross-tenant role stored as `tenant_rns = '*'` and
bypasses all per-tenant checks.

---

## Sub-Documents

- [Concepts](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6655505015/RBAC+%E2%80%94+Concepts) — role definitions, role hierarchy, role
  resolution, RequireRole middleware pattern
- [Implementation](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6655505026/RBAC+%E2%80%94+Implementation) — scope, status table, endpoint-role
  mapping, sequence diagrams, verification
- [Design](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6655505032/RBAC+%E2%80%94+UCP+Design) — Role type, resolveUserRole, RequireRole
  middleware, route grouping, platform-admin handling
- [Horizon Core Data API](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6667713470/RBAC+%E2%80%94+Horizon+Core+Data+API) — member and tenant
  identity endpoints, JWT claim approach, Option 2 feasibility findings

---

## Related

- `MCUCP-191` — RBAC implementation
- `MCUCP-192` — Multi-tenancy (prerequisite: tenant labels on XRs)
