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

Replaces the linear `Role int` hierarchy with an orthogonal permission bitmask.
UCP uses three roles: `developer`, `tenant-admin`, `platform-admin`.
`developer` has `PermProvision` but not `PermApprove` — a developer cannot
approve their own provisioning request. The bitmask is kept for future-proofing:
new roles (e.g. `drift-operator`) can be added with one line in `RolePermissions`.

| Component | File | Status |
|---|---|---|
| 3-role model: `RoleDeveloper`, `RoleTenantAdmin`, `RolePlatformAdmin` (remove `RoleViewer`, `RoleApprover`, `RoleDeployer`) | `auth/context.go` | Not deployed |
| `Permission` bitmask type + `RolePermissions` map | `auth/context.go` | Not deployed |
| `loadRoles(r, tenantID)` — branches on whether tenant is known: `GetRolesForUserInTenant` (≤2 rows) vs `GetAllRolesForUser` (full scan) | `rbac_handler.go` | Not deployed |
| DB method `GetRolesForUserInTenant(userID, tenantRNS)` — `WHERE tenant_rns IN ($tenantRNS, '*')` | `db/roles.go` | Not deployed |
| `RequirePermission(perm)` middleware — resolves tenant from `?tenantId=` or `{tenantSlug}` (1 DB call for slug), then targeted role fetch + bitmask check | `rbac_handler.go` | Not deployed |
| Route registrations updated to use permissions | `main.go` | Not deployed |
| `hasPermission(perm)` replacing `hasMinRole(min)` in `useRole()` | `hooks/useRole.ts` | Not deployed |
| All frontend components updated to use `hasPermission` | all components | Not deployed |
| DB migration: update `tenant_role_assignments.role` CHECK constraint to `('developer','tenant-admin','platform-admin')` | `db/` | Not deployed |

### Phase 3 — Tenant onboarding + login sync (not deployed)

| Component | File | Status |
|---|---|---|
| `ucp_registered_tenants` DB table | `db/` | Deployed |
| DB methods — `RegisterTenant`, `IsTenantRegistered`, `GetRegisteredTenants` | `db/registered_tenants.go` | Deployed |
| Tenant registration endpoint `POST /api/v1/tenants/register` | `tenant_handler.go` | Deployed |
| `oc_roles` DB table — stores OC tenant role + service roles (JSONB) per user per tenant | `db/` | Deployed |
| DB methods — `UpsertOCRoles`, `DeleteOCRoles`, `GetOCRolesForUser` | `db/roles.go` | Deployed |
| Login-triggered OC sync in `CallbackHandler` — UPSERT `oc_roles`, auto-assign `tenant-admin` for OC Admins, revoke for removed members, preserve manually-granted UCP roles (currently Horizon-based; planned replacement: parse JWT `groups` directly — see open question #6) | `bff_auth.go` | Deployed |
| `GET /api/v1/me/tenants` — OC tenants with UCP registration status, UCP role, and admin contact info | `tenant_handler.go` | Deployed |
| Tenant page — register flow + member picker from Horizon for role assignment | frontend | Not deployed |
| Onboarding landing page — 4-state tenant rendering (registered+role / registered+no-role / unregistered+admin / unregistered+member) | frontend | Not deployed |
| ~~Background sync job~~ | — | Deferred — notes only (login sync is sufficient for PoC) |

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
- 3-role model: `developer`, `tenant-admin`, `platform-admin` (scrap `viewer`, `approver`, `deployer`)
- Replace linear `Role int` hierarchy with orthogonal `Permission` bitmask for future-proofing
- `developer` has `PermProvision` but not `PermApprove` — a developer cannot approve their own request
- `RolePermissions` map defines each role as a set of permissions
- All route registrations use `RequirePermission(perm)` instead of `RequireRole(minRole)`
- `loadRoles` branches: targeted `GetRolesForUserInTenant` (≤2 rows) when tenant is known, full `GetAllRolesForUser` only when absent — avoids scanning all tenant memberships for users in many tenants
- `RequirePermission` resolves tenant from `?tenantId=` (GET) or `{tenantSlug}` (mutations, 1 extra DB call) before the role fetch
- Frontend `hasPermission(perm)` replaces `hasMinRole(min)` throughout
- DB migration: update `CHECK` constraint to the 3 new role names

**Phase 3 — Tenant onboarding + login-triggered sync**
- **Tenant registration** — `ucp_registered_tenants` table; a tenant must be explicitly registered by an OC Tenant Admin before UCP operations are allowed
- **`oc_roles` table** — populated on every login with each user's OC tenant role and service roles (JSONB). Not used for access control today; wired up for [Option 2](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6645566515/UCP+Identity+Tenancy+Roles) (runtime OC service-role check) when needed
- **Login-triggered OC sync** — on every OIDC callback, UCP calls Horizon `GET /v0/members/{email}/tenants`, UPSERTs `oc_roles` with current OC data, and for `tenant_role_assignments`: auto-assigns `tenant-admin` if OC Admin (upgrade only), preserves manually-granted roles for OC Members, revokes for removed members
- **`GET /api/v1/me/tenants`** — returns the user's OC tenants enriched with UCP registration status, UCP role, and tenant-admin contact info (name + email) for tenants where the user has no role or the tenant is unregistered
- **Tenant page** — register flow + OC member picker from Horizon for role assignment (no manual email input)
- **Onboarding landing page** — 4-state rendering per tenant: registered+role / registered+no-role (contact admin) / unregistered+OC-admin (register action) / unregistered+OC-member (contact admin)
- **Periodic sync** — deferred, notes only; login sync is sufficient for PoC

### Out of Scope

- **Keycloak configuration** — no changes to Keycloak; roles are owned entirely by UCP.
- **OC role validation at request time** — UCP does not call Core Data on every API request to verify OC standing; consistency is maintained through the login-triggered sync instead.

---

## Open Questions

### 1. Can a tenant-admin grant UCP roles to users who are not OC members?

**How role management works in the PoC:**

On every login, the sync fetches all members of every tenant the logged-in
user belongs to (`GET /v0/tenants/{rns}/members`) and updates UCP's local
data. The role management UI shows this locally-synced member list. A
tenant-admin assigns roles by picking from that list — only OC Tenant Members
of the tenant appear, so non-members cannot be assigned a role through the
normal UI flow.

`AssignRole` itself does not make a live Horizon call; it relies on the
member already having a UCP user record (created on first OIDC login).

**The open question:**

Should a tenant-admin be able to **add a user to the tenant from UCP** — i.e.
invite someone who is not yet an OC Tenant Member? Currently the answer is no:
membership must be managed in OC Portal first, and UCP picks it up on the next
login sync. The question is whether to keep this constraint or allow UCP to
manage OC tenant membership directly via the Core Data API.

The PoC keeps the OC Portal as the source of truth for tenant membership.
Adding a user to a tenant from UCP is out of scope.

---

### 2. Service-level roles for public cloud resources

OneCloud defines per-service roles (DBaaS `admin`/`operator`/`viewer`, CaaS
`admin`/`edit`/`view`, etc.). The PM's Confluence page (UCP Identity, Tenancy
& Roles) only defines OC service roles, not an equivalent UCP service-role layer.

For **OC resources**, two options exist (PM's open question):
- **Option 1 — UCP tenant role only:** `developer` in UCP is sufficient to
  provision any OC service the tenant is subscribed to, regardless of the
  user's OC service-level roles.
- **[Option 2](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6645566515/UCP+Identity+Tenancy+Roles) — UCP role + OC service-role check:** UCP additionally verifies
  the user's OC service role per resource type at provisioning time.

For **public cloud resources (GCP and future providers)**, there is no OC
service-role concept. Access is controlled solely by the UCP tenant role
(`developer` or `tenant-admin`). A developer can provision any resource type
the tenant's credentials cover, with no per-service restriction.

A UCP-native service-role layer (independent of OC) is not defined. Whether
UCP should introduce per-service roles (e.g. `database-operator`) for both OC
and public cloud is an open question to be resolved with the PM before any
such system is designed.

---

### 3. Environment boundary in OneCloud — same tenant or separate tenants?

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

### 4. Impact of OneCloud access revocation on UCP

**Partially addressed in Phase 3.** The login-triggered sync revokes UCP role
assignments for users removed from OC. However there is a window — from when
the OC change happens until the user next logs in — during which the revoked
member retains their UCP access.

Remaining open question: whether this login-gap is acceptable or whether
near-realtime revocation requires the periodic sync (see deferred notes) or
OC webhook integration if it becomes available.

---

### 5. OC role validation at request time vs sync-based consistency

A user could hold UCP `tenant-admin` but have been downgraded to `Tenant Member`
in OC. The login-triggered sync catches this on next login, but during
that window the user retains elevated UCP access.

The Keycloak JWT `groups` claim (see Design doc — Keycloak JWT Structure) makes
[Option 2](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6645566515/UCP+Identity+Tenancy+Roles)
feasible without a live Horizon call: the access token already contains the
user's current OC standing. Middleware could parse `groups` from the token on
each request to verify the user's OC role has not been downgraded. This would
be near-instant revocation at negligible cost (JWT parsing, no external call).

The PoC chooses sync-on-login as the simpler approach. JWT-based request-time
validation is a future improvement.

---

### 6. JWT `groups` format for OC Tenant Members

The observed JWT `groups` claim for a Tenant Admin contains
`rns:roc:iam::{tenant}:roles:admin`. The assumption is that Tenant Members
have `rns:roc:iam::{tenant}:roles:member`. This needs verification with a
non-admin account.

If confirmed, `fetchOCMemberTenants` (the first Horizon call in the login sync)
can be replaced with `parseOCGroupsFromJWT` — the logged-in user's own tenant
membership, tenant role, and all service roles are read directly from the token.

Note: `GET /v0/tenants/{rns}/members` (the second Horizon call) is still needed
to sync OTHER members' data. The JWT only encodes the currently authenticated
user — no other user's roles are available in it.

See the Design doc (Keycloak JWT Structure — JWT-Based Sync) for the planned
implementation once the member `groups` format is confirmed.

---

## Endpoint-to-Role Mapping

| Endpoint | Method | Minimum role | Description |
|---|---|---|---|
| `/api/v1/databases` | GET | `developer` | List databases scoped to caller's tenants |
| `/api/v1/databases` | POST | `developer` | Provision a new database |
| `/api/v1/databases/{name}` | GET | `developer` | Get database details by name |
| `/api/v1/databases/{name}` | DELETE | `developer` | Delete a database |
| `/api/v1/compute` | GET | `developer` | List compute instances scoped to caller's tenants |
| `/api/v1/compute` | POST | `developer` | Provision a new compute instance |
| `/api/v1/compute/{name}` | GET | `developer` | Get compute instance details by name |
| `/api/v1/compute/{name}` | DELETE | `developer` | Delete a compute instance |
| `/api/v1/storage` | GET | `developer` | List storage buckets scoped to caller's tenants |
| `/api/v1/storage` | POST | `developer` | Provision a new storage bucket |
| `/api/v1/storage/{name}` | GET | `developer` | Get storage bucket details by name |
| `/api/v1/storage/{name}` | DELETE | `developer` | Delete a storage bucket |
| `/api/v1/kubernetes-clusters` | GET | `developer` | List Kubernetes clusters scoped to caller's tenants |
| `/api/v1/kubernetes-clusters` | POST | `developer` | Provision a new Kubernetes cluster |
| `/api/v1/kubernetes-clusters/{name}` | GET | `developer` | Get Kubernetes cluster details by name |
| `/api/v1/kubernetes-clusters/{name}` | DELETE | `developer` | Delete a Kubernetes cluster |
| `/api/v1/load-balancers` | GET | `developer` | List load balancer attachments scoped to caller's tenants |
| `/api/v1/load-balancers` | POST | `developer` | Create a new load balancer attachment |
| `/api/v1/load-balancers/{name}` | GET | `developer` | Get load balancer attachment details by name |
| `/api/v1/load-balancers/{name}` | DELETE | `developer` | Delete a load balancer attachment |
| `/api/v1/workflows/{id}/approve` | POST | `tenant-admin` | Approve a pending Temporal workflow |
| `/api/v1/workflows/{id}/reject` | POST | `tenant-admin` | Reject a pending Temporal workflow |
| `/api/v1/drift` | GET | `tenant-admin` | List drift detections for the tenant |
| `/api/v1/quota` | GET | `tenant-admin` | View GCP quota usage for the tenant |
| `/api/v1/gcp/discover` | GET | `tenant-admin` | Discover unmanaged GCP resources for the tenant |
| `/api/v1/settings/api-exposure` | GET, PUT | `tenant-admin` | Read or configure API exposure settings |
| `/api/v1/settings/credentials` | GET, POST | `tenant-admin` | Read or upload GCP credentials for the tenant |
| `/api/v1/settings/credentials/{provider}` | DELETE | `tenant-admin` | Remove a provider's credentials for the tenant |
| `/api/v1/settings/credentials/roc` | GET, POST | `tenant-admin` | Read or configure ROC (Omnia) credentials for the tenant |
| `/api/v1/settings/credentials/all` | GET | `platform-admin` | List all credentials across all tenants |
| `/api/v1/admin/tenants/{tenantSlug}/roles` | GET | `tenant-admin` | List role assignments for a tenant |
| `/api/v1/admin/tenants/{tenantSlug}/roles` | POST | `tenant-admin` | Assign a role to a user within a tenant |
| `/api/v1/admin/tenants/{tenantSlug}/roles/{userID}` | DELETE | `tenant-admin` | Revoke a user's role within a tenant |

---

## UI-to-Role Mapping

### Sidebar navigation

Navigation items are hidden when the caller lacks the minimum role. Direct URL
access (bookmark, address bar) is also blocked — a `RequireRole` route wrapper
in the frontend renders a 403 page instead of the content if the permission
check fails.

| Item | Route | Minimum role | If insufficient role |
|---|---|---|---|
| Database → List | `/` | `developer` | — (all tenant-authenticated users) |
| Database → Create | `/create` | `developer` | Hidden in sidebar + 403 on direct access |
| Compute → List | `/compute` | `developer` | — |
| Compute → Create | `/compute/create` | `developer` | Hidden + 403 |
| Storage → List | `/storage` | `developer` | — |
| Storage → Create | `/storage/create` | `developer` | Hidden + 403 |
| Kubernetes → List | `/kubernetes` | `developer` | — |
| Kubernetes → Create | `/kubernetes/create` | `developer` | Hidden + 403 |
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
| Create button | All resource list pages | `developer` |
| Delete button | All resource list/detail pages | `developer` |
| Approve button | Workflows page | `tenant-admin` |
| Reject button | Workflows page | `tenant-admin` |
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
    DB-->>BFF: [{tenant_rns: "rns:roc:iam::clsd-ucp", role: "developer"},<br/>{tenant_rns: "*", role: "platform-admin"}]

    BFF->>BFF: build roles map<br/>isPlatformAdmin = any row has tenant_rns = "*"
    BFF-->>AuthProvider: {id, email, name,<br/>roles: {"rns:roc:iam::clsd-ucp": "developer"},<br/>isPlatformAdmin: false}

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
    useRole->>useRole: user.isPlatformAdmin? → false<br/>role = user.roles["rns:roc:iam::clsd-ucp"] → "developer"

    useRole-->>Component: {role: "developer", hasPermission}

    Note over Component: hasPermission("read")      → true  → list shown<br/>hasPermission("provision") → true  → Create button shown<br/>hasPermission("approve")   → false → Approve button hidden<br/>hasPermission("manage")    → false → Credentials menu hidden
```

---

## API Sequence — RequirePermission Check (create)

`POST /api/v1/databases` (PermProvision required)

`POST` create requests carry `tenantId` in the request body, not the URL.
The middleware cannot read it, so `loadRoles` falls back to a full scan.
This is acceptable — creates are infrequent compared to reads.

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Session as SessionMiddleware
    participant Middleware as RequirePermission(PermProvision)
    participant DB as PostgreSQL
    participant Handler as CreateDatabase

    Browser->>Session: POST /api/v1/databases<br/>Cookie: session=...<br/>Body: {name, tenantId: "rns:roc:iam::clsd-ucp", ...}
    Session->>Session: decrypt access_token → inject Principal{UserID, ...}
    Session->>Middleware: request with Principal in context

    Note over Middleware: No ?tenantId= or {tenantSlug} — full scan
    Middleware->>DB: GetAllRolesForUser(Principal.UserID)
    DB-->>Middleware: {"rns:roc:iam::clsd-ucp": "developer"}
    Note over Middleware: stored in request context

    Middleware->>Middleware: roleFromMap(roles, "") → max role = RoleDeveloper<br/>RolePermissions[RoleDeveloper] = PermRead|PermProvision<br/>Has(PermProvision)? → true → pass

    Middleware->>Handler: inject RoleDeveloper into context
    Handler-->>Browser: 202 Accepted
```

---

## API Sequence — RequirePermission Check (delete)

`DELETE /api/v1/tenants/clsd-ucp/storage/my-bucket` (PermProvision required)

Slug is resolved to RNS (1 DB call) enabling a targeted role fetch (≤2 rows).
Tenant ownership is enforced by the K8s label filter in the handler — no
separate role check needed inside the handler.

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Session as SessionMiddleware
    participant Middleware as RequirePermission(PermProvision)
    participant DB as PostgreSQL
    participant K8s as Kubernetes
    participant Handler as DeleteStorageBucket

    Browser->>Session: DELETE /api/v1/tenants/clsd-ucp/storage/my-bucket<br/>Cookie: session=...
    Session->>Session: inject Principal
    Session->>Middleware: request with Principal in context

    Middleware->>DB: resolveTenantIDBySlug("clsd-ucp")
    DB-->>Middleware: "rns:roc:iam::clsd-ucp"

    Middleware->>DB: GetRolesForUserInTenant(Principal.UserID, "rns:roc:iam::clsd-ucp")
    DB-->>Middleware: {"rns:roc:iam::clsd-ucp": "developer"}
    Note over Middleware: stored in request context

    Middleware->>Middleware: RolePermissions[RoleDeveloper] = PermRead|PermProvision<br/>Has(PermProvision)? → true → pass

    Middleware->>Handler: request with RoleDeveloper in context
    Handler->>K8s: LIST XObjectStorage<br/>FieldSelector: metadata.name=my-bucket<br/>LabelSelector: platform.ucp.io/tenant=rns-roc-iam--clsd-ucp
    K8s-->>Handler: [{...}] or []
    Handler-->>Browser: 200 OK or 404
```

---

## API Sequence — platform-admin Cross-Tenant Access

`GET /api/v1/databases` (PermRead required, no ?tenantId=)

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Session as SessionMiddleware
    participant Middleware as RequirePermission(PermRead)
    participant DB as PostgreSQL
    participant Handler as ListDatabases

    Browser->>Session: GET /api/v1/databases<br/>Cookie: session=... (no ?tenantId=)
    Session->>Session: decrypt access_token → inject Principal
    Session->>Middleware: request with Principal in context

    Note over Middleware: No ?tenantId= — full scan
    Middleware->>DB: GetAllRolesForUser(Principal.UserID)
    DB-->>Middleware: {"*": "platform-admin"}
    Note over Middleware: stored in request context

    Middleware->>Middleware: roleFromMap(roles, "") → "*" key = platform-admin → RolePlatformAdmin (3)<br/>RolePermissions[RolePlatformAdmin] has all perms → Has(PermRead)? → true → pass

    Middleware->>Handler: request with RolePlatformAdmin in context
    Handler->>Handler: platform-admin bypasses tenant scope<br/>returns all tenants' resources
    Handler-->>Browser: 200 OK
```

---

## Verification

### developer blocked from approve and credentials

```bash
COOKIE="Cookie: session=<developer-session>"

# List — allowed
curl -s -H "$COOKIE" "http://localhost:8080/api/v1/databases?tenantId=$TENANT" | jq '.count'
# → 200

# Create — allowed
curl -s -X POST -H "$COOKIE" \
  -d '{"name":"test","provider":"gcp","engineVersion":"15","projectId":"...","tenantId":"'$TENANT'"}' \
  http://localhost:8080/api/v1/databases
# → 202

# Approve — blocked
curl -s -X POST -H "$COOKIE" \
  http://localhost:8080/api/v1/tenants/clsd-ucp/workflows/<id>/approve
# {"error":"insufficient permission: approve required"}

# Credentials — blocked
curl -s -X POST -H "$COOKIE" \
  http://localhost:8080/api/v1/settings/credentials
# {"error":"insufficient permission: manage required"}
```

### tenant-admin can approve and provision

```bash
COOKIE="Cookie: session=<tenant-admin-session>"

# Approve — allowed
curl -s -X POST -H "$COOKIE" \
  http://localhost:8080/api/v1/tenants/clsd-ucp/workflows/<id>/approve
# → 200

# Create resource — allowed
curl -s -X POST -H "$COOKIE" \
  -d '{"name":"test","provider":"gcp",...}' \
  http://localhost:8080/api/v1/storage
# → 202
```

### cross-tenant access blocked

```bash
# User is tenant-admin of clsd-ucp but not other-tenant
curl -s -H "$COOKIE" \
  "http://localhost:8080/api/v1/databases?tenantId=rns:roc:iam::other-tenant"
# {"error":"insufficient permission: read required"}  (or 403)
```
