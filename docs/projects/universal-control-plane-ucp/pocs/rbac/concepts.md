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

## Role Resolution via Horizon

Role membership is resolved against the Horizon API, consistent with the
existing `isUserTenantAdmin()` pattern. The Horizon members endpoint returns
the caller's tenant subscriptions, each of which carries a `default_role` field:

```
GET /v0/members/<email>/tenants?subscriptions=true
Authorization: Bearer {access_token}

Response:
{
  "items": [
    {
      "rns": "rns:roc:iam::clsd-ucp",
      "subscriptions": [
        { "default_role": "tenant-admin", "service": "ucp" }
      ]
    }
  ]
}
```

`resolveUserRole(r, tenantID)` calls this endpoint, finds the entry matching
`tenantID`, reads `subscriptions[].default_role`, and maps it to the internal
`Role` type.

### Role Mapping

| Horizon `default_role` | Internal `Role` |
|---|---|
| `platform-admin` | `RolePlatformAdmin` |
| `tenant-admin` | `RoleTenantAdmin` |
| `deployer` | `RoleDeployer` |
| `approver` | `RoleApprover` |
| `viewer` | `RoleViewer` |
| absent / unrecognised | `RoleUnknown` (→ 403) |

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

Resolution: the Horizon members endpoint returns all tenants for a user.
A user whose membership record carries `default_role: platform-admin` on any
tenant entry is identified as a platform admin. Alternatively, a dedicated
Horizon platform group may be checked — the exact mechanism is determined
by what the Horizon API exposes.

---

## Relation to isUserTenantAdmin

`isUserTenantAdmin()` from MCUCP-192 is a binary check — admin or not. MCUCP-191
replaces it with `RequireRole` at the route level. All endpoints currently guarded
by `isUserTenantAdmin()` are migrated to `RequireRole(RoleTenantAdmin)` or a more
appropriate minimum role as a starting point.

`isUserTenantAdmin()` is retained only for the delete ownership check path in
MCUCP-192, where it is called with the XR's annotation tenant rather than the
request's `X-Tenant-ID`. That specific use is unchanged.
