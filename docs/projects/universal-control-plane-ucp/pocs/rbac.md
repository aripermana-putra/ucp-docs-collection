---
title: "RBAC"
space: UCP
parent_page_id: "../pocs.md"
---

# RBAC

UCP enforces access control at the API server layer using a
**5-role per-tenant model**. Each role defines the minimum set of operations
a user can perform within a tenant. Role resolution is delegated to the
Horizon API, consistent with the existing `isUserTenantAdmin()` pattern
from MCUCP-192.

---

## Approach Summary

Every authenticated request passes through a `RequireRole(minRole)` middleware
before reaching the handler. The middleware resolves the caller's role for the
tenant in `X-Tenant-ID` by calling the Horizon members API and maps the
returned subscription role to the internal `Role` type. Requests that do not
meet the minimum required role are rejected with HTTP 403.

`platform-admin` is a cross-tenant role that bypasses per-tenant checks. It is
identified via a dedicated Horizon group membership check.

---

## Sub-Documents

- [Concepts](rbac/concepts.md) — role definitions, role hierarchy, Horizon role
  resolution, RequireRole middleware pattern
- [Implementation](rbac/implementation.md) — scope, status table, endpoint-role
  mapping, sequence diagrams, verification
- [Design](rbac/ucp-rbac-design.md) — Role type, resolveUserRole, RequireRole
  middleware, route grouping, platform-admin handling

---

## Related

- `MCUCP-191` — RBAC implementation
- `MCUCP-192` — Multi-tenancy (prerequisite: tenant labels on XRs)
- `docs/architecture/RBAC.md` — role model specification
