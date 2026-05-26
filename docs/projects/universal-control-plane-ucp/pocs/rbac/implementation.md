---
title: "RBAC — Implementation"
space: UCP
parent_page_id: "../rbac.md"
---

# RBAC — Implementation

---

## Implementation

| Component | File | Status |
|---|---|---|
| `Role` type + context helpers | `auth/context.go` | Deployed |
| `tenant_role_assignments` DB schema + migration | `db/` | Deployed |
| DB methods — `GetTenantRole`, `GetAllRolesForUser`, `GetRoleAssignmentsForTenant`, `AssignTenantRole`, `RevokeTenantRole` | `db/roles.go` | Deployed |
| `db.FindUserByEmail()` (for AssignRole email → user_id lookup) | `db/roles.go` | Deployed |
| `resolveUserRole()` | `rbac_handler.go` | Deployed |
| `RequireRole(minRole)` middleware | `rbac_handler.go` | Deployed |
| Admin API handlers — `ListRoleAssignments`, `AssignRole`, `RevokeRole` | `rbac_handler.go` | Deployed |
| Route grouping by role level + admin API route registration | `main.go` | Deployed |
| Remove `isUserTenantAdmin()` from create/list resource handlers | all resource handlers | Deployed |
| Remove `isUserTenantAdmin()` from settings handlers (credentials) | `settings_handler.go` | Deployed |
| `/auth/me` role extension | `bff_auth.go` (`MeHandler`) | Deployed |
| `useRole()` hook | `hooks/useRole.ts`, frontend | Deployed |
| Sidebar role-aware rendering (hide nav items) | `Sidebar.jsx` | Deployed |
| In-page action buttons role-aware rendering | all list components | Deployed |
| `ForbiddenPage` component | `pages/ForbiddenPage.jsx` | Deployed |
| `RequireRole` route wrapper — wrap restricted routes in `App.jsx` | `App.jsx` | Deployed |
| Role management UI | `pages/RoleManagementPage.jsx` | Deployed |

---

## Scope

### In Scope

- **Role type** — ordered `Role` integer type in `auth/context.go` enabling `>=` comparison
- **DB schema** — `tenant_role_assignments(user_id, tenant_rns, role)` table; `platform-admin` stored as `tenant_rns = '*'`
- **Role resolution** — `resolveUserRole()` queries `tenant_role_assignments` using `Principal.UserID` from the request context (injected by `SessionMiddleware`)
- **Admin API** — list, assign, and revoke role assignments per tenant; gated behind `tenant-admin`
- **`/auth/me` role extension** — `MeHandler` queries `tenant_role_assignments` and adds a `roles` map and `isPlatformAdmin` flag to the response; frontend reads this on mount
- **`useRole()` hook** — when a tenant is selected, returns the role for that tenant; when no tenant is selected, returns the user's maximum role across all tenants (mirrors backend `roleFromMap` behaviour so sidebar and page access reflect the highest role anywhere)
- **DB query methods** — `GetTenantRole`, `GetAllRolesForUser`, `AssignTenantRole`, `RevokeTenantRole`, `GetRoleAssignmentsForTenant`, `FindUserByEmail`
- **Admin API** — `ListRoleAssignments`, `AssignRole`, `RevokeRole` handlers + route registration; gated behind `tenant-admin`
- **RequireRole middleware** — per-route middleware that resolves role, enforces minimum, and injects resolved role into request context
- **Route refactoring** — routes grouped by minimum role; `isUserTenantAdmin()` removed from all resource and settings handlers and replaced by middleware
- **platform-admin bypass** — cross-tenant role resolved independently of `X-Tenant-ID`
- **Sidebar role-aware rendering** — nav items hidden when caller lacks minimum role
- **In-page action buttons** — create/delete hidden for viewer and approver; approve/reject hidden below approver
- **`RequireRole` route wrapper + `ForbiddenPage`** — direct URL access blocked with 403 page for insufficient role
- **Role management UI** — frontend page for admins to manage role assignments within a tenant

### Out of Scope

- **Keycloak configuration** — no changes to Keycloak; roles are owned entirely by UCP.

---

## Open Questions

### 1. Environment boundary in OneCloud — same tenant or separate tenants?

In OneCloud, does a single tenant span multiple deployment environments
(e.g. staging and production), or is each environment represented as a
distinct tenant with its own membership and service subscriptions?

This matters for UCP because:
- If staging and production are **separate OC tenants**, the existing
  per-tenant isolation is sufficient — a user's access to each environment
  is naturally gated by their OC membership in each tenant.
- If staging and production exist **within the same OC tenant**, UCP must
  introduce its own per-environment scoping layer. This would affect the
  `tenant_role_assignments` schema (adding an `environment` column), the
  ProviderConfig naming convention, and how credentials are registered.

The current PoC assumes one UCP context per OC tenant and does not model
environments as a separate dimension.

---

### 2. Multiple cloud provider projects per tenant

The PoC operates on the assumption that one OC tenant maps to exactly
one GCP project (or one project per public cloud provider). It is unclear
whether a team may have multiple GCP projects under the same OC tenant —
for example, one project per service or one project per environment.

If multiple projects per tenant are required:
- The current `ProviderConfig` naming convention
  (`gcp-{sanitized-tenant-id}-{env}`) assumes a single project and would
  need to be extended.
- The "UCP Project" entity proposed by the PM would need to model a
  one-to-many relationship between an OC tenant and its cloud projects.

---

### 3. Impact of OneCloud access revocation on UCP

UCP stores its own role assignments in `tenant_role_assignments`,
independent of OC membership. If a user's access is revoked in OC
(removed from a tenant, or the tenant itself is deleted), UCP is not
automatically notified and the user's UCP roles remain active.

Questions to resolve:
- Should UCP periodically sync membership from OC and remove stale UCP
  role assignments?
- Should UCP validate OC membership on every request (adding a Horizon
  API call to the hot path)?
- Or is it acceptable that a tenant-admin manually revokes the UCP role
  before or after the OC revocation?

The PoC does not implement any sync mechanism. The current behaviour is
that OC revocation has no immediate effect on UCP access.

---

### 4. Service account credential lifecycle and key rotation

UCP stores GCP (and OC) service account credentials in Kubernetes Secrets
(PoC) or GCP Secret Manager (production target). No key rotation mechanism
is implemented.

Questions to resolve:
- When a service account key expires or is rotated externally, Crossplane
  will start failing reconciliation for that tenant's resources with
  authentication errors. Does UCP surface this clearly to the tenant-admin,
  or does it silently leave resources in an error state?
- Should UCP implement automatic key rotation (e.g. via ESO + GCP Secret
  Manager rotation policies), or is manual update via the credentials
  endpoint (`POST /api/v1/settings/credentials`) sufficient?
- What is the expected SLA for a tenant-admin to respond to a broken
  credential — can provisioning/reconciliation be paused rather than
  failing?

The PoC treats credentials as static — no rotation logic is implemented.
Stale credentials will cause Crossplane reconciliation errors that require
manual credential re-upload to resolve.

---

## Endpoint-to-Role Mapping

| Endpoint | Method | Minimum role | Description |
|---|---|---|---|
| `/api/v1/databases` | GET | `viewer` | List databases scoped to caller's tenants |
| `/api/v1/databases` | POST | `deployer` | Provision a new database |
| `/api/v1/databases/{name}` | GET | `viewer` | Get database details by name |
| `/api/v1/databases/{name}` | DELETE | `deployer` | Delete a database |
| `/api/v1/compute` | GET | `viewer` | List compute instances scoped to caller's tenants |
| `/api/v1/compute` | POST | `deployer` | Provision a new compute instance |
| `/api/v1/compute/{name}` | GET | `viewer` | Get compute instance details by name |
| `/api/v1/compute/{name}` | DELETE | `deployer` | Delete a compute instance |
| `/api/v1/storage` | GET | `viewer` | List storage buckets scoped to caller's tenants |
| `/api/v1/storage` | POST | `deployer` | Provision a new storage bucket |
| `/api/v1/storage/{name}` | GET | `viewer` | Get storage bucket details by name |
| `/api/v1/storage/{name}` | DELETE | `deployer` | Delete a storage bucket |
| `/api/v1/kubernetes-clusters` | GET | `viewer` | List Kubernetes clusters scoped to caller's tenants |
| `/api/v1/kubernetes-clusters` | POST | `deployer` | Provision a new Kubernetes cluster |
| `/api/v1/kubernetes-clusters/{name}` | GET | `viewer` | Get Kubernetes cluster details by name |
| `/api/v1/kubernetes-clusters/{name}` | DELETE | `deployer` | Delete a Kubernetes cluster |
| `/api/v1/load-balancers` | GET | `viewer` | List load balancer attachments scoped to caller's tenants |
| `/api/v1/load-balancers` | POST | `deployer` | Create a new load balancer attachment |
| `/api/v1/load-balancers/{name}` | GET | `viewer` | Get load balancer attachment details by name |
| `/api/v1/load-balancers/{name}` | DELETE | `deployer` | Delete a load balancer attachment |
| `/api/v1/workflows/{id}/approve` | POST | `approver` | Approve a pending Temporal workflow |
| `/api/v1/workflows/{id}/reject` | POST | `approver` | Reject a pending Temporal workflow |
| `/api/v1/drift` | GET | `tenant-admin` | List drift detections for the tenant |
| `/api/v1/quota` | GET | `tenant-admin` | View GCP quota usage for the tenant |
| `/api/v1/gcp/discover` | GET | `tenant-admin` | Discover unmanaged GCP resources for the tenant |
| `/api/v1/settings/api-exposure` | GET, PUT | `tenant-admin` | Read or configure API exposure settings |
| `/api/v1/settings/credentials` | GET, POST | `tenant-admin` | Read or upload GCP credentials for the tenant |
| `/api/v1/settings/credentials/{provider}` | DELETE | `tenant-admin` | Remove a provider's credentials for the tenant |
| `/api/v1/settings/credentials/roc` | GET, POST | `tenant-admin` | Read or configure ROC (Omnia) credentials for the tenant |
| `/api/v1/settings/credentials/all` | GET | `platform-admin` | List all credentials across all tenants |
| `/api/v1/admin/tenants/{tenantRNS}/roles` | GET | `tenant-admin` | List role assignments for a tenant |
| `/api/v1/admin/tenants/{tenantRNS}/roles` | POST | `tenant-admin` | Assign a role to a user within a tenant |
| `/api/v1/admin/tenants/{tenantRNS}/roles/{userID}` | DELETE | `tenant-admin` | Revoke a user's role within a tenant |

---

## UI-to-Role Mapping

### Sidebar navigation

Navigation items are hidden when the caller lacks the minimum role. Direct URL
access (bookmark, address bar) is also blocked — a `RequireRole` route wrapper
renders a 403 page instead of the content if the check fails.

| Item | Route | Minimum role | If insufficient role |
|---|---|---|---|
| Database → List | `/` | `viewer` | — (all tenant-authenticated users) |
| Database → Create | `/create` | `deployer` | Hidden in sidebar + 403 on direct access |
| Compute → List | `/compute` | `viewer` | — |
| Compute → Create | `/compute/create` | `deployer` | Hidden + 403 |
| Storage → List | `/storage` | `viewer` | — |
| Storage → Create | `/storage/create` | `deployer` | Hidden + 403 |
| Kubernetes → List | `/kubernetes` | `viewer` | — |
| Kubernetes → Create | `/kubernetes/create` | `deployer` | Hidden + 403 |
| Drift → List | `/drift` | `tenant-admin` | Hidden + 403 |
| Quotas → Usage | `/quota` | `tenant-admin` | Hidden + 403 |
| Terraform | `/terraform/*` | — | Out of RBAC scope (experimental) |
| Admin → Workflows | `/admin/workflows` | `platform-admin` | Hidden + 403 |
| Admin → Kube Resources | `/admin/kube-resources` | `platform-admin` | Hidden + 403 |
| Settings → Credentials | `/settings/credentials` | `tenant-admin` | Hidden + 403 |
| Settings → Tenants | `/admin/tenants` | `platform-admin` | Hidden + 403 |
| Settings → Role Management | `/settings/roles` | `tenant-admin` | Hidden + 403 |

### In-page actions

| Action | Page | Minimum role |
|---|---|---|
| Create button | All resource list pages | `deployer` |
| Delete button | All resource list/detail pages | `deployer` |
| Approve button | Workflows page | `approver` |
| Reject button | Workflows page | `approver` |
| Upload credentials | Settings → Credentials | `tenant-admin` |
| Assign / revoke role | Settings → Role Management | `tenant-admin` |

---

## API Sequence — /auth/me on Mount

The frontend `AuthProvider` calls `/auth/me` on mount. With the role extension,
the response now includes the caller's role assignments so the UI has everything
it needs before the first page renders.

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant AuthProvider as AuthProvider.jsx
    participant BFF as API Server
    participant DB as PostgreSQL

    Browser->>AuthProvider: app mounts
    AuthProvider->>BFF: GET /auth/me<br/>Cookie: session=...

    BFF->>BFF: SessionMiddleware → inject Principal{UserID, ...}
    BFF->>DB: SELECT role, tenant_rns FROM tenant_role_assignments<br/>WHERE user_id = Principal.UserID
    DB-->>BFF: [{tenant_rns: "rns:roc:iam::clsd-ucp", role: "deployer"},<br/>{tenant_rns: "*", role: "platform-admin"}]

    BFF->>BFF: build roles map<br/>isPlatformAdmin = any row has tenant_rns = "*"
    BFF-->>AuthProvider: {id, email, name,<br/>roles: {"rns:roc:iam::clsd-ucp": "deployer"},<br/>isPlatformAdmin: false}

    AuthProvider->>AuthProvider: store user{roles, isPlatformAdmin} in context
    Note over AuthProvider: isAuthenticated = true<br/>user.roles available to all components
```

---

## UI Sequence — Role-Aware Rendering

When the user selects a tenant, `useRole()` derives the effective role from the
already-stored `user.roles` — no additional API call.

```mermaid
sequenceDiagram
    autonumber

    participant User
    participant TenantSelector as TenantSelector
    participant TenantCtx as TenantContext
    participant Component as DatabasesPage
    participant useRole as useRole()

    User->>TenantSelector: select "clsd-ucp" tenant
    TenantSelector->>TenantCtx: setSelectedTenant({rns: "rns:roc:iam::clsd-ucp"})
    TenantCtx->>Component: Outlet re-mounts (key changed)

    Component->>useRole: useRole()
    useRole->>useRole: user.isPlatformAdmin? → false<br/>role = user.roles["rns:roc:iam::clsd-ucp"] → "deployer"

    useRole-->>Component: {role: "deployer", hasMinRole}

    Note over Component: hasMinRole("viewer")      → true  → list shown<br/>hasMinRole("deployer")    → true  → Create button shown<br/>hasMinRole("approver")    → false → Approve button hidden<br/>hasMinRole("tenant-admin")→ false → Credentials menu hidden
```

---

## API Sequence — RequireRole Check (create)

`POST /api/v1/databases` (deployer minimum, X-Tenant-ID present)

`RequireRole` calls `loadRoles` first — **1 DB call** fetches all role
assignments and caches them in the request context.

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Session as SessionMiddleware
    participant Middleware as RequireRole("deployer")
    participant DB as PostgreSQL
    participant Handler as CreateDatabase

    Browser->>Session: POST /api/v1/databases<br/>Cookie: session=...<br/>X-Tenant-ID: rns:roc:iam::clsd-ucp
    Session->>Session: decrypt access_token → inject Principal{UserID, ...}
    Session->>Middleware: request with Principal in context

    Middleware->>DB: GetAllRolesForUser(Principal.UserID)
    DB-->>Middleware: {"rns:roc:iam::clsd-ucp": "deployer", ...}
    Note over Middleware: cache stored in request context

    Middleware->>Middleware: roleFromMap(cache, "rns:roc:iam::clsd-ucp") → RoleDeployer (3)<br/>RoleDeployer (3) >= RoleDeployer (3) → pass

    Middleware->>Handler: inject RoleDeployer + cache into context
    Handler-->>Browser: 202 Accepted

    note over Middleware: viewer (1) < deployer (3) → 403<br/>tenant-admin (4) >= deployer (3) → pass
```

---

## API Sequence — RequireRole Check (delete with ownership)

`DELETE /api/v1/storage/my-bucket` (deployer minimum, no X-Tenant-ID)

Cache populated in middleware; ownership check in handler reads from cache —
**1 DB call total** for the entire request.

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Session as SessionMiddleware
    participant Middleware as RequireRole("deployer")
    participant DB as PostgreSQL
    participant Handler as DeleteStorageBucket

    Browser->>Session: DELETE /api/v1/storage/my-bucket<br/>Cookie: session=... (no X-Tenant-ID)
    Session->>Session: inject Principal
    Session->>Middleware: request with Principal in context

    Middleware->>DB: GetAllRolesForUser(Principal.UserID)
    DB-->>Middleware: {"rns:roc:iam::clsd-ucp": "deployer", ...}
    Note over Middleware: cache stored in request context

    Middleware->>Middleware: roleFromMap(cache, "") → max role = RoleDeployer<br/>RoleDeployer >= RoleDeployer → pass

    Middleware->>Handler: request with cache in context
    Handler->>Handler: read platform.ucp.io/tenant-id from XR annotation<br/>xrTenantID = "rns:roc:iam::clsd-ucp"
    Handler->>Handler: resolveUserRole(r, xrTenantID) → reads cache (0 DB calls)<br/>roleFromMap(cache, "rns:roc:iam::clsd-ucp") = RoleDeployer
    Handler->>Handler: RoleDeployer >= RoleDeployer → authorized
    Handler-->>Browser: 200 OK
```

---

## API Sequence — platform-admin Cross-Tenant Access

`GET /api/v1/databases` (viewer minimum, no X-Tenant-ID)

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Session as SessionMiddleware
    participant Middleware as RequireRole("viewer")
    participant DB as PostgreSQL
    participant Handler as ListDatabases

    Browser->>Session: GET /api/v1/databases<br/>Cookie: session=... (no X-Tenant-ID)
    Session->>Session: decrypt access_token → inject Principal
    Session->>Middleware: request with Principal in context

    Middleware->>DB: GetAllRolesForUser(Principal.UserID)
    DB-->>Middleware: {"*": "platform-admin"}
    Note over Middleware: cache stored in request context

    Middleware->>Middleware: roleFromMap(cache, "") → "*" key = platform-admin → RolePlatformAdmin (5)<br/>RolePlatformAdmin (5) >= RoleViewer (1) → pass

    Middleware->>Handler: request with RolePlatformAdmin + cache in context
    Handler->>Handler: platform-admin bypasses tenant scope<br/>returns all tenants' resources
    Handler-->>Browser: 200 OK
```

---

## Verification

### viewer blocked from POST

```bash
COOKIE="Cookie: session=<viewer-session>"

# List — allowed
curl -s -H "$COOKIE" -H "X-Tenant-ID: $TENANT" \
  http://localhost:8080/api/v1/databases | jq '.count'
# → 200

# Create — blocked
curl -s -X POST -H "$COOKIE" -H "X-Tenant-ID: $TENANT" \
  -d '{"name":"test","provider":"gcp","engineVersion":"15","projectId":"..."}' \
  http://localhost:8080/api/v1/databases
# {"error":"insufficient role: deployer required"}
```

### deployer blocked from credentials

```bash
COOKIE="Cookie: session=<deployer-session>"

curl -s -X POST -H "$COOKIE" -H "X-Tenant-ID: $TENANT" \
  http://localhost:8080/api/v1/settings/credentials
# {"error":"insufficient role: tenant-admin required"}
```

### approver can approve, not provision

```bash
COOKIE="Cookie: session=<approver-session>"

# Approve — allowed
curl -s -X POST -H "$COOKIE" -H "X-Tenant-ID: $TENANT" \
  http://localhost:8080/api/v1/workflows/<id>/approve
# → 200

# Create resource — blocked
curl -s -X POST -H "$COOKIE" -H "X-Tenant-ID: $TENANT" \
  -d '{"name":"test","provider":"gcp",...}' \
  http://localhost:8080/api/v1/storage
# {"error":"insufficient role: deployer required"}
```
