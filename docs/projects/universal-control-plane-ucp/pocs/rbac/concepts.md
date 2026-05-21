---
title: "RBAC — Concepts"
space: UCP
parent_page_id: "../rbac.md"
---

# RBAC — Concepts

---

## The 5-Role Model

| Role | Scope | Description |
|---|---|---|
| `platform-admin` | Platform | UCP operators. Full access across all tenants with no tenant-level checks. |
| `tenant-admin` | Tenant | Full access within their tenant, including credential management. |
| `deployer` | Tenant | Provision and delete resources within their tenant. |
| `approver` | Tenant | Approve or reject Temporal approval workflows. |
| `viewer` | Tenant | Read-only access to resources and status. |

Roles are ordered by privilege. A user with a higher role automatically satisfies
checks for all lower roles:

```
platform-admin > tenant-admin > deployer > approver > viewer
```

A `RequireRole("deployer")` check passes for `tenant-admin` and `platform-admin` as
well — not only for `deployer`.

---

## Role Resolution

`resolveUserRole(r, tenantID)` is called by `RequireRole` on every protected
request. By the time it runs, `SessionMiddleware` has already executed and
injected the `Principal` into the request context. `Principal.UserID` is the
internal DB UUID used to look up the caller's role:

```go
roleStr, _ := s.db.GetTenantRole(principal.UserID, tenantID)
```

Role assignments are stored in a `tenant_role_assignments` table in the
existing PostgreSQL database. Role changes take effect immediately. Roles are
managed via a dedicated admin API and the role management UI.

## Role Management

Roles are assigned and revoked through the API and a role management page in
the UI. The endpoints are gated behind `tenant-admin` for tenant-scoped
operations and `platform-admin` for cross-tenant operations:

| Endpoint | Method | Minimum role | Description |
|---|---|---|---|
| `/api/v1/admin/tenants/{tenantRNS}/roles` | GET | `tenant-admin` | List role assignments for a tenant |
| `/api/v1/admin/tenants/{tenantRNS}/roles` | POST | `tenant-admin` | Assign a role to a user |
| `/api/v1/admin/tenants/{tenantRNS}/roles/{userID}` | DELETE | `tenant-admin` | Remove a role assignment |

`platform-admin` can access these endpoints across all tenants regardless of
`X-Tenant-ID`.

---

## Role Type and Hierarchy

The `Role` type is an integer so that permission checks reduce to a single
comparison:

```go
// auth/context.go
type Role int

const (
    RoleUnknown     Role = iota // 0
    RoleViewer                  // 1
    RoleApprover                // 2
    RoleDeployer                // 3
    RoleTenantAdmin             // 4
    RolePlatformAdmin           // 5
)
```

A `RequireRole(minRole)` check is `userRole >= minRole`.

---

## RequireRole Middleware

`RequireRole(minRole Role)` is a per-route middleware. It:

1. Reads `X-Tenant-ID` from the request header.
2. Calls `resolveUserRole(r, tenantID)` to get the caller's role.
3. If `userRole >= minRole` → injects the resolved role into the request context
   and calls the next handler.
4. Otherwise → returns 403.

`platform-admin` bypasses step 1 — it is resolved independently of `X-Tenant-ID`
via a Horizon platform group membership check.

The resolved role is available to handlers via `RoleFromContext(r.Context())`.
Handlers no longer call `isUserTenantAdmin()` directly.

---

## platform-admin

`platform-admin` is a platform-scoped role, not tied to any individual tenant.
It grants access to every endpoint, including cross-tenant list operations
(list all resources regardless of `X-Tenant-ID`).

Resolution follows the same mechanism as tenant-scoped roles — either a
dedicated `platform-admin` value in the Keycloak JWT claim or a DB entry
with `tenant_rns = '*'` (or similar sentinel). The exact representation
depends on the chosen approach and is defined in `ucp-rbac-design.md`.

---

## Relation to isUserTenantAdmin

`isUserTenantAdmin()` from MCUCP-192 is a binary check — admin or not. MCUCP-191
replaces it with `RequireRole` at the route level. All endpoints currently guarded
by `isUserTenantAdmin()` are migrated to `RequireRole(RoleTenantAdmin)` or a more
appropriate minimum role as a starting point.

`isUserTenantAdmin()` is retained only for the delete ownership check path in
MCUCP-192, where it is called with the XR's annotation tenant rather than the
request's `X-Tenant-ID`. That specific use is unchanged.
