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

- [Concepts](rbac/concepts.md) — role definitions, role hierarchy, role
  resolution, RequireRole middleware pattern
- [Implementation](rbac/implementation.md) — scope, status table, endpoint-role
  mapping, sequence diagrams, verification
- [Design](rbac/ucp-rbac-design.md) — Role type, resolveUserRole, RequireRole
  middleware, route grouping, platform-admin handling
- [Horizon Core Data API](rbac/horizon-core-data-api.md) — member and tenant
  identity endpoints, JWT claim approach, Option 2 feasibility findings

---

## Related

- `MCUCP-191` — RBAC implementation
- `MCUCP-192` — Multi-tenancy (prerequisite: tenant labels on XRs)
- `docs/architecture/RBAC.md` — role model specification
