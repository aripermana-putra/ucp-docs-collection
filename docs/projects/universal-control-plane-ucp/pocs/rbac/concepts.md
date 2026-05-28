---
title: "RBAC — Concepts"
space: UCP
parent_page_id: "../rbac.md"
---

# RBAC — Concepts

---

## The 3-Role Model

| Role | Scope | Description |
|---|---|---|
| `platform-admin` | Platform | UCP operators. Full access across all tenants with no tenant-level checks. |
| `tenant-admin` | Tenant | Full access within their tenant — provision, approve/reject, credentials, role management. |
| `developer` | Tenant | Provision and delete resources within their tenant. Cannot approve workflows. |

Roles are expressed as a **permission bitmask** rather than a linear integer. Each
role is a set of permissions:

| Permission | `developer` | `tenant-admin` | `platform-admin` |
|---|:---:|:---:|:---:|
| `read` (list, get) | ✅ | ✅ | ✅ |
| `provision` (create, delete) | ✅ | ✅ | ✅ |
| `approve` (approve/reject workflows) | ❌ | ✅ | ✅ |
| `manage` (credentials, settings, roles) | ❌ | ✅ | ✅ |
| `platform` (cross-tenant operations) | ❌ | ❌ | ✅ |

A `developer` has `provision` but not `approve` — they cannot approve their own
provisioning request. A `tenant-admin` has both. Adding a new role in the future
is one line in the `RolePermissions` map with no schema changes.

---

## Role Resolution

`resolveUserRole(r, tenantID)` is called by `RequirePermission` on every
protected request. By the time it runs, `SessionMiddleware` has already
executed and injected the `Principal` into the request context.
`Principal.UserID` is the internal DB UUID used to look up all role
assignments for the caller in one query:

```go
allRoles, _ := s.db.GetAllRolesForUser(principal.UserID)
// returns map[tenantRNS]roleString for all tenants the user has a role in
```

The result is cached in the request context — at most one DB call per request,
regardless of how many permission checks happen downstream.

Role assignments are stored in the `tenant_role_assignments` table in the
existing PostgreSQL database. Role changes take effect on the user's next login.
Roles are managed via a dedicated admin API and the role management UI.

## Role Management

Roles are assigned and revoked through the API and a role management page in
the UI. The endpoints are gated behind `tenant-admin` for tenant-scoped
operations and `platform-admin` for cross-tenant operations:

| Endpoint | Method | Minimum role | Description |
|---|---|---|---|
| `/api/v1/admin/tenants/{tenantSlug}/roles` | GET | `tenant-admin` | List role assignments for a tenant |
| `/api/v1/admin/tenants/{tenantSlug}/roles` | POST | `tenant-admin` | Assign a role to a user |
| `/api/v1/admin/tenants/{tenantSlug}/roles/{userID}` | DELETE | `tenant-admin` | Remove a role assignment |

`platform-admin` can access these endpoints across all tenants regardless of
which tenant is in scope.

---

## Role Type and Permission Model

The `Role` type is still an integer constant but enforcement uses a permission
bitmask rather than `>=` comparison:

```go
// auth/context.go
type Role int

const (
    RoleUnknown     Role = iota // 0 — unresolved / not a tenant member
    RoleDeveloper               // 1
    RoleTenantAdmin             // 2
    RolePlatformAdmin           // 3
)

type Permission uint

const (
    PermRead      Permission = 1 << 0
    PermProvision Permission = 1 << 1
    PermApprove   Permission = 1 << 2
    PermManage    Permission = 1 << 3
    PermPlatform  Permission = 1 << 4
)

var RolePermissions = map[string]Permission{
    "developer":      PermRead | PermProvision,
    "tenant-admin":   PermRead | PermProvision | PermApprove | PermManage,
    "platform-admin": PermRead | PermProvision | PermApprove | PermManage | PermPlatform,
}
```

A `RequirePermission(PermApprove)` check resolves the caller's role, looks up
its `Permission` set, and passes only if `permissions.Has(PermApprove)`. A
`developer` fails this check; `tenant-admin` and `platform-admin` pass.

---

## RequirePermission Middleware

`RequirePermission(perm Permission)` is a per-route middleware. It:

1. Reads `tenantId` from the `?tenantId=` query param (GET requests) or
   `{tenantSlug}` path segment (mutations).
2. Calls `loadRoles(r)` — fetches all role assignments for the caller from DB
   and caches them in the request context (at most one DB call per request).
3. Calls `resolveUserRole(r, tenantID)` to derive the caller's `Role` for
   that tenant.
4. Maps `Role` → `Permission` set via `RolePermissions`.
5. If `permissions.Has(perm)` → injects the resolved role into the request
   context and calls the next handler.
6. Otherwise → returns 403.

`platform-admin` has all permissions (`PermPlatform` included) and passes
every check regardless of tenant scope.

The resolved role is available to handlers via `RoleFromContext(r.Context())`.
Handlers do not call `isUserTenantAdmin()` — all access enforcement is at the
middleware layer.

---

## platform-admin

`platform-admin` is a platform-scoped role, not tied to any individual tenant.
It holds all permissions (`PermRead | PermProvision | PermApprove | PermManage | PermPlatform`)
and passes every `RequirePermission` check regardless of which tenant is in scope.

Resolution uses the same DB mechanism as tenant-scoped roles — a row in
`tenant_role_assignments` with `tenant_rns = '*'`. `roleFromMap` checks for
this sentinel row first before performing the tenant-scoped lookup.

`platform-admin` is assigned and revoked manually only — it is never touched
by the login-triggered OC sync.

---

## OC Roles

During the login-triggered sync, UCP writes the user's raw OC roles to an
`oc_roles` table — one row per user per tenant:

```
oc_roles
  user_id          UUID
  tenant_rns       TEXT
  oc_tenant_role   TEXT    -- 'Tenant Admin' or 'Tenant Member'
  oc_service_roles JSONB   -- {"DBaaS": "operator", "CaaS": "admin", ...}
  synced_at        TIMESTAMPTZ
```

This table is **not used for access control today**. UCP's `RequirePermission`
checks read only from `tenant_role_assignments`. This table exists so that if
runtime OC service-role validation (Confluence Option 2) is ever enabled, the
data is already present — no additional Horizon calls or schema changes are
required. Wiring it up is a middleware change only.

---

## Tenant Onboarding and Role Seeding

When a user logs in via OIDC, UCP synchronises their role assignments against
their current OC standing:

- **OC Tenant Admin** → automatically assigned `tenant-admin` in UCP (upgrade
  only — never downgraded if already a higher role)
- **OC Tenant Member** → no UCP role assigned automatically; a UCP
  `tenant-admin` must explicitly grant `developer` or `tenant-admin`
- **Removed from OC tenant** → UCP role for that tenant is revoked on next login
- Manually-granted UCP roles (e.g. an OC Member promoted to `developer`) are
  always preserved by the sync

A user with no UCP role can still log in and see their tenants. They cannot
take any action until a tenant-admin grants them a role.

---

## Role Management

Roles are assigned and revoked through the API and a role management page in
the UI. The role management endpoints are gated behind `tenant-admin`:

| Endpoint | Method | Minimum permission | Description |
|---|---|---|---|
| `/api/v1/admin/tenants/{tenantSlug}/roles` | GET | `manage` | List role assignments for a tenant |
| `/api/v1/admin/tenants/{tenantSlug}/roles` | POST | `manage` | Assign a role to a user |
| `/api/v1/admin/tenants/{tenantSlug}/roles/{userID}` | DELETE | `manage` | Remove a role assignment |

The role management UI presents the OC tenant member list fetched from
Horizon, so a `tenant-admin` can assign roles by selecting from the list
rather than typing email addresses manually.
