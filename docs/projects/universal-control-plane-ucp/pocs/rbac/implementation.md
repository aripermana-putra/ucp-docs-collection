---
title: "RBAC — Implementation"
space: UCP
parent_page_id: "../rbac.md"
---

# RBAC — Implementation

---

## Implementation

### Phase 1 — Core RBAC (deployed)

| Component | File | Status |
|---|---|---|
| `Role` type + context helpers | `auth/context.go` | Deployed |
| `tenant_role_assignments` DB schema + migration | `db/` | Deployed |
| DB methods — `GetTenantRole`, `GetAllRolesForUser`, `GetRoleAssignmentsForTenant`, `AssignTenantRole`, `RevokeTenantRole` | `db/roles.go` | Deployed |
| `db.FindUserByEmail()` | `db/roles.go` | Deployed |
| `resolveUserRole()` with per-request cache | `rbac_handler.go` | Deployed |
| `RequireRole(minRole)` middleware | `rbac_handler.go` | Deployed |
| Admin API — `ListRoleAssignments`, `AssignRole`, `RevokeRole` | `rbac_handler.go` | Deployed |
| Route grouping by role level + admin API route registration | `main.go` | Deployed |
| Remove `isUserTenantAdmin()` from all handlers | all resource + settings handlers | Deployed |
| `/auth/me` role extension | `bff_auth.go` | Deployed |
| `useRole()` hook | `hooks/useRole.ts` | Deployed |
| Sidebar role-aware rendering | `Sidebar.jsx` | Deployed |
| In-page action buttons role-aware rendering | all list components | Deployed |
| `ForbiddenPage` + `RequireRole` route wrapper | `App.jsx`, frontend | Deployed |
| Role management UI | `pages/RoleManagementPage.jsx` | Deployed |

### Phase 2 — Permission model refactor (not deployed)

Replaces the linear role hierarchy with an orthogonal permission-based model.
`deployer` and `approver` become parallel — a deployer cannot approve their
own request.

| Component | File | Status |
|---|---|---|
| `Permission` bitmask type + `RolePermissions` map | `auth/context.go` | Not deployed |
| `RequirePermission(perm)` middleware replacing `RequireRole` | `rbac_handler.go` | Not deployed |
| Route registrations updated to use permissions | `main.go` | Not deployed |
| Delete handler ownership check updated to use permissions | all resource handlers | Not deployed |
| `hasPermission(perm)` replacing `hasMinRole(min)` in `useRole()` | `hooks/useRole.ts` | Not deployed |
| All frontend components updated to use `hasPermission` | all components | Not deployed |

### Phase 3 — Tenant onboarding + user seeding + sync (not deployed)

| Component | File | Status |
|---|---|---|
| `ucp_registered_tenants` DB table | `db/` | Not deployed |
| DB methods — `RegisterTenant`, `IsRegistered`, `GetRegisteredTenants` | `db/tenants.go` | Not deployed |
| Tenant registration endpoint `POST /api/v1/tenants/register` | `tenant_handler.go` | Not deployed |
| User seeding on first login (Core Data → `tenant_role_assignments`) | `bff_auth.go` (`CallbackHandler`) | Not deployed |
| Tenant member list from Core Data on login | `horizon_handler.go` | Not deployed |
| Background sync job — polls Core Data, updates `tenant_role_assignments` | `sync/` | Not deployed |
| `GET /api/v1/me/tenants` — user's OC tenants with UCP status | `tenant_handler.go` | Not deployed |
| Onboarding landing page — tenant status per tenant | frontend | Not deployed |

---

## Scope

### In Scope

**Phase 1 — Core RBAC (deployed)**
- Role type, DB schema, role resolution with per-request cache
- `RequireRole` middleware replacing all `isUserTenantAdmin()` per-handler checks
- Admin API to manage role assignments manually
- `/auth/me` extended with roles map and `isPlatformAdmin` flag
- Role-aware frontend — sidebar, page buttons, route guards, role management UI

**Phase 2 — Permission model refactor**
- Replace linear `Role int` hierarchy with orthogonal `Permission` bitmask
- `deployer` and `approver` become parallel — deployer cannot approve; approver cannot provision
- `RolePermissions` map defines each role as a set of permissions
- All route registrations use `RequirePermission(perm)` instead of `RequireRole(minRole)`
- Frontend `hasPermission(perm)` replaces `hasMinRole(min)` throughout

**Phase 3 — Tenant onboarding + user seeding + sync**
- **Tenant registration** — `ucp_registered_tenants` table; a tenant must be explicitly registered in UCP by an OC Tenant Admin before any UCP operations are allowed for it
- **User seeding on first login** — on successful OIDC callback, UCP calls Core Data `GET /v0/members/{email}/tenants?subscriptions=true`, fetches all registered tenants the user belongs to, and seeds UCP roles based on their OC standing (OC Tenant Admin → UCP `tenant-admin`; OC Tenant Member → UCP `viewer`)
- **Tenant member seeding on first admin login** — when an OC Tenant Admin registers a tenant, UCP fetches all current OC members of that tenant via `GET /v0/tenants/{tenantRNS}/members` and seeds their UCP roles
- **Periodic sync** — background polling job calls Core Data at a configurable interval, compares current OC membership against `tenant_role_assignments`, adds new members, and removes or downgrades stale assignments
- **OC → UCP role mapping** — explicit mapping used by seeding and sync (see Design doc)
- **`GET /api/v1/me/tenants`** — returns the user's OC tenants enriched with UCP registration status and their UCP role per tenant
- **Onboarding landing page** — shows per-tenant status: registered + has role / registered + no role ("contact your tenant-admin") / not registered in UCP

### Out of Scope

- **Keycloak configuration** — no changes to Keycloak; roles are owned entirely by UCP.
- **OC role validation at request time** — UCP does not call Core Data on every API request to verify OC standing; consistency is maintained through seeding and periodic sync instead.

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

**Partially addressed in Phase 3.** The periodic sync job detects OC membership
removals and removes the corresponding UCP role assignments. However there is a
window of up to the sync interval (default 15 min) during which a revoked OC
member retains their UCP access.

Remaining open question: whether the sync interval is acceptable or whether
near-realtime revocation requires a different approach (e.g. OC webhook
integration if it becomes available, or a shorter polling interval for
high-sensitivity tenants).

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

### 5. OC role validation at request time vs sync-based consistency

A user could hold UCP `tenant-admin` but have been downgraded to `Tenant Member`
in OC. The Phase 3 sync catches this within the polling interval, but during
that window the user retains elevated UCP access.

Should UCP validate the user's current OC standing on every protected request
(PM's Option 2)? This would make access revocation near-instant but adds a
Core Data API call to every hot path.

The PoC chooses sync-based consistency (polling) as the simpler approach.
Request-time OC validation is deferred and would require the OC → UCP role
mapping to be applied at the middleware layer rather than only at seed/sync time.

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
