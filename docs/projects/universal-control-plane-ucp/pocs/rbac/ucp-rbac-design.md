---
title: "RBAC — UCP Design"
space: UCP
parent_page_id: "../rbac.md"
---

# RBAC — UCP Design

---

## Permission-Based Role Model (Phase 2)

UCP uses three roles: `developer`, `tenant-admin`, and `platform-admin`.
Rather than a linear integer hierarchy, each role is defined as a set of
**permissions** using a bitmask. The enforcement check changes from
`role >= minRole` to `role.Has(permission)`.

This ensures future roles can be added (e.g. a `drift-operator` that can
approve drift alerts but not provision resources) without restructuring the
hierarchy or renumbering constants.

```go
// auth/context.go
type Permission uint

const (
    PermRead      Permission = 1 << 0  // list, get resources
    PermProvision Permission = 1 << 1  // create, delete resources
    PermApprove   Permission = 1 << 2  // approve, reject workflows
    PermManage    Permission = 1 << 3  // credentials, settings, role management
    PermPlatform  Permission = 1 << 4  // cross-tenant operations
)

var RolePermissions = map[string]Permission{
    "developer":      PermRead | PermProvision,
    "tenant-admin":   PermRead | PermProvision | PermApprove | PermManage,
    "platform-admin": PermRead | PermProvision | PermApprove | PermManage | PermPlatform,
}

func (p Permission) Has(required Permission) bool {
    return p&required == required
}
```

`developer` has `PermProvision` but not `PermApprove` — a developer cannot
approve their own provisioning request. Approval is `tenant-admin` only.

Adding a new role in the future (e.g. `drift-operator: PermRead | PermApprove | PermDrift`)
is one line in `RolePermissions` — no schema change, no hierarchy restructuring.

### RequirePermission middleware

```go
func (s *APIServer) RequirePermission(perm authpkg.Permission) mux.MiddlewareFunc {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            r, _, err = s.loadRoles(r)
            userPerms := resolvedPermissions(r, tenantID)  // derives Permission from role string
            if !userPerms.Has(perm) {
                respondError(w, http.StatusForbidden,
                    fmt.Sprintf("insufficient permission: %s required", perm))
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

Route registrations become self-documenting:

```go
api.Handle("/databases",              server.requirePermHandler(authpkg.PermRead,      ...)).Methods("GET")
api.Handle("/databases",              server.requirePermHandler(authpkg.PermProvision,  ...)).Methods("POST")
api.Handle("/workflows/{id}/approve", server.requirePermHandler(authpkg.PermApprove,   ...)).Methods("POST")
api.Handle("/settings/credentials",   server.requirePermHandler(authpkg.PermManage,    ...)).Methods("GET")
api.Handle("/settings/credentials/all", server.requirePermHandler(authpkg.PermPlatform,...)).Methods("GET")
```

### Frontend `hasPermission`

```ts
const ROLE_PERMISSIONS: Record<Role, Set<string>> = {
  'developer':      new Set(['read', 'provision']),
  'tenant-admin':   new Set(['read', 'provision', 'approve', 'manage']),
  'platform-admin': new Set(['read', 'provision', 'approve', 'manage', 'platform']),
}

// useRole() exposes hasPermission instead of hasMinRole
{hasPermission('provision') && <CreateButton />}
{hasPermission('approve')   && <ApproveButton />}
{hasPermission('manage')    && <CredentialsMenu />}
```

---

## Role Type (Phase 1 — deployed)

```go
// auth/context.go
type Role int

const (
    RoleUnknown     Role = iota // 0 — unresolved / not a tenant member
    RoleDeveloper               // 1
    RoleTenantAdmin             // 2
    RolePlatformAdmin           // 3
)

func (r Role) String() string {
    switch r {
    case RoleDeveloper:     return "developer"
    case RoleTenantAdmin:   return "tenant-admin"
    case RolePlatformAdmin: return "platform-admin"
    default:                return "unknown"
    }
}

type roleContextKey struct{}

func WithRole(ctx context.Context, role Role) context.Context {
    return context.WithValue(ctx, roleContextKey{}, role)
}

func RoleFromContext(ctx context.Context) (Role, bool) {
    r, ok := ctx.Value(roleContextKey{}).(Role)
    return r, ok
}
```

---

## Database ERD

```mermaid
erDiagram
    identity_providers {
        UUID id PK
        TEXT name
        TEXT issuer_url
        TEXT client_id
        TEXT jwks_uri
        TEXT env
        BOOLEAN is_active
        TIMESTAMPTZ created_at
    }

    users {
        UUID id PK
        UUID idp_id FK
        TEXT external_id
        TEXT email
        TEXT username
        TEXT display_name
        TEXT status
        TIMESTAMPTZ last_login_at
        INT login_count
        TIMESTAMPTZ created_at
    }

    sessions {
        TEXT id PK
        UUID user_id FK
        TEXT access_token
        TEXT refresh_token
        TIMESTAMPTZ expires_at
        TIMESTAMPTZ created_at
        TIMESTAMPTZ last_seen_at
        TEXT ip_address
        TEXT user_agent
    }

    audit_logs {
        UUID id PK
        UUID user_id FK
        TEXT session_id
        TEXT action
        TEXT resource
        JSONB metadata
        TIMESTAMPTZ created_at
    }

    tenant_role_assignments {
        TEXT user_id FK
        TEXT tenant_rns
        TEXT role
        TIMESTAMPTZ created_at
    }

    oc_roles {
        UUID user_id FK
        TEXT tenant_rns
        TEXT oc_tenant_role
        JSONB oc_service_roles
        TIMESTAMPTZ synced_at
    }

    identity_providers ||--o{ users : "idp_id"
    users ||--o{ sessions : "user_id"
    users ||--o{ audit_logs : "user_id"
    users ||--o{ tenant_role_assignments : "user_id"
    users ||--o{ oc_roles : "user_id"
```

| Table | Origin | Modified by MCUCP-191 |
|---|---|---|
| `identity_providers` | Pre-existing | No |
| `users` | Pre-existing | No |
| `sessions` | Pre-existing | No |
| `audit_logs` | Pre-existing | No |
| `tenant_role_assignments` | **Added by MCUCP-191** | — |
| `oc_roles` | **Added by MCUCP-191** | — |

No pre-existing tables were altered. MCUCP-191 adds `tenant_role_assignments`
(UCP's own access control) and `oc_roles` (a persistent mirror of OC
role data populated during the login sync).

`tenant_rns = '*'` in `tenant_role_assignments` denotes a platform-admin — a
cross-tenant role not bound to any specific tenant RNS.

---

## Database Schema

```sql
CREATE TABLE tenant_role_assignments (
    user_id    TEXT NOT NULL REFERENCES users(id),
    tenant_rns TEXT NOT NULL,
    role       TEXT NOT NULL CHECK (role IN (
                   'platform-admin', 'tenant-admin', 'developer')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, tenant_rns)
);

CREATE TABLE oc_roles (
    user_id         UUID NOT NULL REFERENCES users(id),
    tenant_rns      TEXT NOT NULL,
    -- OC Level 1 tenant role: 'Tenant Admin' or 'Tenant Member'
    oc_tenant_role  TEXT NOT NULL,
    -- OC Level 2 service roles: {"DBaaS": "operator", "CaaS": "admin", ...}
    -- Null when service roles have not been fetched for this tenant yet.
    oc_service_roles JSONB,
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, tenant_rns)
);
```

`tenant_role_assignments` is UCP's own access-control table — rows here
drive all `RequirePermission` checks. `platform-admin` is stored with
`tenant_rns = '*'`, not tied to any specific tenant.

`oc_roles` is a persistent mirror of OC data populated during the
login-triggered sync. It is not used for access control today. When Option 2
(runtime OC role check) is enabled, the permission middleware reads from this
table instead of (or in addition to) calling the Horizon API at request time.

---

## Role Resolution — Design

Role resolution uses a **per-request cache** to avoid repeated DB queries.
A single `GetAllRolesForUser` call is made at most once per request; all
subsequent checks within the same request read from the Go context.

### loadRoles

`loadRoles` fetches all role assignments for the caller and stores them in
the request context. Returns immediately on cache hit.

```go
func (s *APIServer) loadRoles(r *http.Request) (*http.Request, map[string]string, error) {
    if cached, ok := r.Context().Value(cachedRolesKey{}).(map[string]string); ok {
        return r, cached, nil  // cache hit — 0 DB calls
    }
    roles, err := s.db.GetAllRolesForUser(principal.UserID)  // 1 DB call
    r = r.WithContext(context.WithValue(r.Context(), cachedRolesKey{}, roles))
    return r, roles, nil
}
```

### roleFromMap

Derives a `Role` from the cached map for a given `tenantID`:

```go
func roleFromMap(allRoles map[string]string, tenantID string) Role {
    if allRoles["*"] == "platform-admin" {
        return RolePlatformAdmin            // platform-admin sentinel
    }
    if tenantID != "" {
        return stringToRole(allRoles[tenantID])  // specific tenant
    }
    // ?tenantId= absent — return highest role across all tenants
    max := RoleUnknown
    for _, roleStr := range allRoles {
        if r := stringToRole(roleStr); r > max { max = r }
    }
    return max
}
```

When `?tenantId=` is absent, the caller's maximum role across all their
tenants is used. This allows `RequirePermission` checks to pass without
requiring the param — for example, a user with `developer` in tenant-A can
delete a resource without specifying `?tenantId=`. The label-based K8s
ownership filter in the handler verifies the resource belongs to a tenant
the user has the required role in.

### resolveUserRole

Reads from context cache — zero additional DB calls after `loadRoles` has run:

```go
func (s *APIServer) resolveUserRole(r *http.Request, tenantID string) (Role, error) {
    allRoles, ok := r.Context().Value(cachedRolesKey{}).(map[string]string)
    if !ok {
        _, allRoles, _ = s.loadRoles(r)  // fallback if cache not populated
    }
    return roleFromMap(allRoles, tenantID), nil
}
```

### DB call count per request

| Scenario | DB calls |
|---|---|
| Any request through `RequireRole` | 1 (`GetAllRolesForUser` in middleware) |
| Delete handler ownership check | 0 (reads context cache) |
| Any subsequent check in same request | 0 (cache hit) |
| `/auth/me` | 1 (`GetAllRolesForUser` directly, independent of cache) |

---

## Admin API

Role assignments are managed via three endpoints, all gated behind
`RequireRole(RoleTenantAdmin)`. `platform-admin` passes all role checks
automatically.

```go
// rbac_handler.go
func (s *APIServer) ListRoleAssignments(w http.ResponseWriter, r *http.Request)
func (s *APIServer) AssignRole(w http.ResponseWriter, r *http.Request)
func (s *APIServer) RevokeRole(w http.ResponseWriter, r *http.Request)
```

`AssignRole` request body:

```json
{ "userEmail": "user@example.com", "role": "developer" }
```

The handler resolves `userEmail` to a `user_id` via the `users` table (user
must have logged in at least once for JIT provisioning to have created their
record) before inserting into `tenant_role_assignments`.

---

## /auth/me Extension

`MeHandler` in `bff_auth.go` is extended to query `tenant_role_assignments`
for the caller and include the results in the response. The frontend calls
this on mount via `AuthProvider`, so role information is available immediately
after login.

Extended response:

```json
{
  "id": "abc123",
  "email": "user@example.com",
  "name": "User Name",
  "roles": {
    "rns:roc:iam::clsd-ucp": "developer",
    "rns:roc:iam::other-tenant": "developer"
  },
  "isPlatformAdmin": false
}
```

`isPlatformAdmin` is `true` when a `tenant_rns = '*'` row exists for the user.
The `roles` map contains only tenant-scoped assignments — `platform-admin` is
not included as a tenant entry, it is expressed via `isPlatformAdmin`.

---

## useRole Hook

`useRole()` derives the caller's effective role for the currently selected
tenant from the data already in `AuthProvider`. No extra API call needed.

```ts
type Role = 'platform-admin' | 'tenant-admin' | 'developer'

function useRole() {
  const { user } = useAuth()
  const { selectedTenant } = useTenantContext()

  if (user?.isPlatformAdmin) {
    return { role: 'platform-admin', hasMinRole: () => true }
  }

  const role: Role | null = selectedTenant
    ? (user?.roles?.[selectedTenant.rns] ?? null)
    : null

  const hasMinRole = (min: Role) =>
    role !== null && ROLE_LEVEL[role] >= ROLE_LEVEL[min]

  return { role, hasMinRole }
}
```

Usage in components:

```tsx
const { hasPermission } = useRole()

{hasPermission('provision') && <CreateButton />}
{hasPermission('provision') && <DeleteButton />}
{hasPermission('approve')   && <ApproveButton />}
{hasPermission('manage')    && <CredentialsMenuItem />}
```

When no tenant is selected, `role` is `null` and `hasPermission` returns `false`
for all checks — all action buttons are hidden until the user selects a tenant.

---

## Frontend Route Guard

Hiding a menu item is not sufficient — a user can still navigate directly to a
restricted URL via the address bar or a bookmark. A `RequireRole` wrapper
component renders a 403 page instead of the route content when the check fails:

```tsx
// components/RequireRole.tsx
function RequireRole({ min, children }: { min: Role; children: ReactNode }) {
  const { hasMinRole } = useRole()

  if (!hasMinRole(min)) {
    return <ForbiddenPage />
  }
  return <>{children}</>
}
```

Applied per route in `App.jsx`:

```tsx
<Route
  path="/settings/credentials"
  element={
    <RequireRole min="tenant-admin">
      <SettingsCredentialsPage />
    </RequireRole>
  }
/>
<Route
  path="/admin/kube-resources"
  element={
    <RequireRole min="platform-admin">
      <KubeResourcesPage />
    </RequireRole>
  }
/>
```

`ForbiddenPage` displays a 403 message and a link back to the home page. It
does not redirect — a redirect would hide the fact that the access was denied,
making it harder to diagnose misconfigured role assignments.

---

## RequireRole Middleware

`RequireRole` calls `loadRoles` first to populate the per-request cache,
then resolves the effective role and enforces the minimum. The handler and
any subsequent `resolveUserRole` calls in the same request hit the cache.

```go
// rbac_handler.go
func (s *APIServer) RequireRole(minRole authpkg.Role) mux.MiddlewareFunc {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            tenantID := strings.TrimSpace(r.URL.Query().Get("tenantId"))

            // Populate cache — at most 1 DB call per request.
            var err error
            r, _, err = s.loadRoles(r)
            if err != nil {
                respondError(w, http.StatusInternalServerError,
                    "Failed to load roles: "+err.Error())
                return
            }

            role, err := s.resolveUserRole(r, tenantID)  // reads cache
            if err != nil {
                respondError(w, http.StatusInternalServerError,
                    "Failed to resolve user role: "+err.Error())
                return
            }
            if role < minRole {
                respondError(w, http.StatusForbidden,
                    fmt.Sprintf("insufficient role: %s required", minRole))
                return
            }

            // Pass updated request (with cache + resolved role in context).
            next.ServeHTTP(w, r.WithContext(authpkg.WithRole(r.Context(), role)))
        })
    }
}
```

---

## Route Registration

Routes are registered with `requirePermHandler(perm, handler)` — a per-route
wrapper that enforces the required permission without subrouters. The
`isUserTenantAdmin()` calls previously in individual handlers are removed.

```go
// main.go — route registration
// Resource reads — PermRead
api.Handle("/databases",              server.requirePermHandler(authpkg.PermRead,      server.ListDatabases)).Methods("GET")
api.Handle("/databases/{name}",       server.requirePermHandler(authpkg.PermRead,      server.GetDatabase)).Methods("GET")
// ... other GET endpoints

// Resource mutations — PermProvision (slug-based delete paths from MCUCP-192)
api.Handle("/databases",              server.requirePermHandler(authpkg.PermProvision,  server.CreateDatabase)).Methods("POST")
api.Handle("/tenants/{tenantSlug}/databases/{name}", server.requirePermHandler(authpkg.PermProvision, server.DeleteDatabase)).Methods("DELETE")
// ... other POST/DELETE endpoints

// Workflow approval — PermApprove
api.Handle("/tenants/{tenantSlug}/workflows/{workflowId}/approve", server.requirePermHandler(authpkg.PermApprove, server.ApproveWorkflow)).Methods("POST")
api.Handle("/tenants/{tenantSlug}/workflows/{workflowId}/reject",  server.requirePermHandler(authpkg.PermApprove, server.RejectWorkflow)).Methods("POST")

// Credential management — PermManage
api.Handle("/settings/credentials", server.requirePermHandler(authpkg.PermManage, server.GetCredentials)).Methods("GET")
api.Handle("/settings/credentials", server.requirePermHandler(authpkg.PermManage, server.UpsertCredentials)).Methods("POST")

// Platform-wide operations — PermPlatform
api.Handle("/settings/credentials/all", server.requirePermHandler(authpkg.PermPlatform, server.ListAllCredentials)).Methods("GET")

// Role management — PermManage (slug resolved to RNS inside handler)
api.Handle("/admin/tenants/{tenantSlug}/roles",          server.requirePermHandler(authpkg.PermManage, server.ListRoleAssignments)).Methods("GET")
api.Handle("/admin/tenants/{tenantSlug}/roles",          server.requirePermHandler(authpkg.PermManage, server.AssignRole)).Methods("POST")
api.Handle("/admin/tenants/{tenantSlug}/roles/{userID}", server.requirePermHandler(authpkg.PermManage, server.RevokeRole)).Methods("DELETE")
```

---

## Handler Cleanup

`RequirePermission` enforces access at the route level. Individual handlers
no longer call `isUserTenantAdmin()` for create operations.

Delete handlers use the label-based K8s ownership check from MCUCP-192 — the
single `List(FieldSelector=name, LabelSelector=tenant)` call both fetches the
resource and verifies tenant ownership in one round-trip. No Horizon call or
per-handler role check is needed for delete operations.

---

## Role Management UI

A role management page in the frontend lets `tenant-admin` and `platform-admin`
users assign and revoke roles within a tenant. The page is accessible from the
Settings or Admin section and scoped to the tenant selected in the global
`TenantSelector`.

The UI calls the admin API endpoints defined above. It needs to:
- List current role assignments for the selected tenant
- Show a user search/input to assign a role (user must have logged in once)
- Allow changing or removing an existing assignment

---

## Tenant Registration (Phase 3)

A tenant must be explicitly registered in UCP before any resources can be
provisioned for it. Registration is performed by an OC Tenant Admin.

### DB schema

```sql
CREATE TABLE ucp_registered_tenants (
    tenant_rns    TEXT NOT NULL PRIMARY KEY,
    registered_by UUID REFERENCES users(id),
    registered_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Registration flow

```
POST /api/v1/tenants/register
Body: { "tenantRNS": "rns:roc:iam::coupon-team" }
Requires: caller is OC Tenant Admin (verified via Core Data GET /v0/tenants/{rns})
```

On successful registration:
1. Insert row into `ucp_registered_tenants`
2. Fetch all OC members of the tenant via `GET /v0/tenants/{tenantRNS}/members`
3. Seed `tenant_role_assignments` for each member (see OC → UCP role mapping below)

---

## OC → UCP Role Mapping

Used during login-triggered sync to derive a UCP role from a user's OC standing:

| OC Level 1 (tenant role) | UCP role assigned |
|---|---|
| Tenant Admin | `tenant-admin` (auto-assigned on login) |
| Tenant Member | none (must be assigned manually by a UCP `tenant-admin`) |

OC Tenant Members do not receive any UCP role automatically. They can log in,
see their tenants, and know whether a tenant is registered in UCP — but they
cannot take any action until a UCP `tenant-admin` grants them the `developer`
or `tenant-admin` role explicitly.

This keeps provisioning permission explicitly controlled by the tenant-admin,
not derived from OC membership level.

---

## Login-Triggered OC Sync

On every successful OIDC login (in `CallbackHandler`), UCP synchronises the
caller's role assignments against their current OC standing. This replaces a
periodic background job for the PoC.

```
1. Call GET /v0/members/{email}/tenants (Horizon)
2. For each tenant in the OC response:
   a. UPSERT oc_roles with oc_tenant_role from OC response.
      Optionally call GET /v0/tenants/{rns}/members/{email} to populate
      oc_service_roles if service-role data is available.
   b. Apply UCP role sync:
      - OC role = Tenant Admin AND current UCP role < tenant-admin (or none)
        → assign tenant-admin in tenant_role_assignments
      - OC role = Tenant Admin AND current UCP role ≥ tenant-admin → no change
      - OC role = Tenant Member → never touch UCP role (preserve any
        manually-granted developer or tenant-admin role)
3. For each tenant where user has a UCP role assignment but is no longer
   present in the OC response → revoke the UCP role; delete from oc_roles
4. platform-admin rows (tenant_rns = '*') are never touched by the sync
```

**Sync rules summary:**
- OC Admin → auto-assign `tenant-admin` (upgrade only, never downgrade)
- OC Member → preserve any existing manually-assigned UCP role
- Removed from OC tenant → revoke UCP role; remove OC roles row
- `platform-admin` → managed manually only, OC sync does not affect it
- **OC roles always written to `oc_roles`** regardless of UCP role outcome

**PoC gap:** if a user's OC role changes between logins, their UCP role
reflects the stale state until their next login. A future periodic sync
(see notes below) closes this window.

---

## `GET /api/v1/me/tenants`

Returns the user's OC tenants enriched with their UCP status per tenant.
Used by the onboarding landing page.

```json
{
  "items": [
    {
      "rns": "rns:roc:iam::coupon-team",
      "name": "coupon-team",
      "ocRole": "Tenant Admin",
      "ucpRegistered": true,
      "ucpRole": "tenant-admin",
      "admins": null
    },
    {
      "rns": "rns:roc:iam::other-team",
      "name": "other-team",
      "ocRole": "Tenant Member",
      "ucpRegistered": true,
      "ucpRole": null,
      "admins": [
        { "name": "Alice Smith", "email": "alice@example.com" }
      ]
    },
    {
      "rns": "rns:roc:iam::new-team",
      "name": "new-team",
      "ocRole": "Tenant Admin",
      "ucpRegistered": false,
      "ucpRole": null,
      "admins": null
    },
    {
      "rns": "rns:roc:iam::locked-team",
      "name": "locked-team",
      "ocRole": "Tenant Member",
      "ucpRegistered": false,
      "ucpRole": null,
      "admins": [
        { "name": "Bob Jones", "email": "bob@example.com" }
      ]
    }
  ]
}
```

`admins` is populated from the OC tenant's admin list whenever the user
has no UCP role or the tenant is not registered — so they know who to
contact. For tenants where the user already has a UCP role, `admins` is
omitted.

The frontend uses this to render three states per tenant:
- `ucpRegistered: true` + `ucpRole` set → full access at that role level
- `ucpRegistered: true` + `ucpRole: null` → "Contact your tenant-admin to get access" — `admins` list from OC is included so the user can see who to contact (name + email)
- `ucpRegistered: false` + OC role = Tenant Admin → "UCP not set up — register this tenant" (register action available)
- `ucpRegistered: false` + OC role = Tenant Member → "UCP not set up for this tenant" — `admins` list shown so the user can request setup

The `admins` field is populated from OC for unregistered tenants and for
registered tenants where the user has no UCP role.

---

## Periodic Sync (deferred — notes only)

A background polling job to keep `tenant_role_assignments` consistent with OC
membership between user logins is deferred for the PoC.

The login-triggered sync handles the same use cases for active users. The gap
is users whose OC role changes while they already have an active UCP session —
their role reflects stale state until their next login.

When implemented, the sync would:
```
For each tenant in ucp_registered_tenants:
  1. GET /v0/tenants/{tenantRNS}/members  (Core Data)
  2. Apply the same login-triggered sync rules:
     - OC Admin with lower UCP role → upgrade to tenant-admin
     - User removed from OC → revoke UCP role
     - OC Member with manually-granted UCP role → preserve
     - platform-admin rows → never touched
```

No event streaming is available from Core Data, so polling would be the only
option. The sync interval and acceptable staleness window are to be defined
when this is implemented.

---

## Open Design Questions

### Service-Level Roles for Public Cloud Resources

OneCloud has a well-defined service-role system per service (DBaaS
`admin`/`operator`/`viewer`, CaaS `admin`/`edit`/`view`, etc.). The PM's
Confluence page (UCP Identity, Tenancy & Roles) documents OC service roles
but does not define an equivalent UCP service-role model.

For **OC resources**, two options exist (PM's open question):
- **Option 1 — UCP tenant role only:** UCP checks only the caller's UCP
  `developer` or `tenant-admin` role. The OC service account executes
  requests on behalf of the team; OC service roles are not checked.
- **Option 2 — UCP role + OC service-role check:** UCP additionally verifies
  the user holds the required OC service role for the resource type being
  provisioned (e.g. DBaaS `operator` to provision a database). Requires a
  UCP-maintained OC-service-to-resource-type mapping.

The `oc_roles` table stores both the OC tenant role and OC service roles
(JSONB) per user per tenant, populated on every login. Enabling Option 2
requires only wiring the permission middleware to read from `oc_roles`
at provisioning time — no additional data collection or schema changes needed.

For **public cloud resources (GCP and future providers)**, there is no
OC service-role concept. Access is controlled solely by the UCP tenant role
(`developer` or `tenant-admin`). A `developer` can provision any resource
type the tenant's credentials cover, with no per-service restriction.

A UCP-native service-role layer (independent of OC) is not defined. Whether
UCP should introduce per-service roles (e.g. `database-operator`) for both
OC and public cloud resources is an open question to be resolved with the PM.
