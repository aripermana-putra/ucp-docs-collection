---
title: "RBAC — Implementation"
space: UCP
parent_page_id: "../rbac.md"
---

# RBAC — Implementation

---

## Implementation

### Phase 1 + 2 — Core RBAC + Permission model (deployed)

| Component | File | Status |
|---|---|---|
| 3-role model: `RoleDeveloper`, `RoleTenantAdmin`, `RolePlatformAdmin` | `auth/context.go` | Deployed |
| `Permission` bitmask type + `RolePermissions` map | `auth/context.go` | Deployed |
| `tenant_role_assignments` DB schema + migration (3-role CHECK constraint) | `db/` | Deployed |
| DB methods — `GetTenantRole`, `GetAllRolesForUser`, `GetRoleAssignmentsForTenant`, `GetRolesForUserInTenant`, `AssignTenantRole`, `RevokeTenantRole` | `db/roles.go` | Deployed |
| `db.FindUserByEmail()` | `db/roles.go` | Deployed |
| `loadRoles(r, tenantID)` — targeted `GetRolesForUserInTenant` (≤2 rows) when tenant known, full scan when absent | `rbac_handler.go` | Deployed |
| `resolveUserRole()` with per-request in-memory store | `rbac_handler.go` | Deployed |
| `RequirePermission(perm)` middleware — resolves tenant from `?tenantId=` or `{tenantSlug}`, bitmask check | `rbac_handler.go` | Deployed |
| `requirePermHandler` per-route permission wrapper | `rbac_handler.go` | Deployed |
| Admin API — `ListRoleAssignments`, `AssignRole`, `RevokeRole` (slug-based paths; self-revoke blocked with 403) | `rbac_handler.go` | Deployed |
| All routes updated to `requirePermHandler(PermXxx, handler)` | `main.go` | Deployed |
| Remove `isUserTenantAdmin()` from all handlers | all resource + settings handlers | Deployed |
| `/auth/me` role extension | `bff_auth.go` | Deployed |
| `useRole()` hook with `hasPermission(perm)` | `hooks/useRole.ts` | Deployed |
| Sidebar permission-aware rendering | `Sidebar.jsx` | Deployed |
| In-page action buttons permission-aware rendering | all list components | Deployed |
| `ForbiddenPage` + `RequireRole` (perm-based) route wrapper | `App.jsx`, frontend | Deployed |
| Role management inline in tenant page (no separate RoleManagementPage) | `TenantInfo.jsx`, frontend | Deployed |

### Phase 3 — Tenant onboarding + login sync

| Component | File | Status |
|---|---|---|
| `ucp_registered_tenants` DB table | `db/` | Deployed |
| DB methods — `RegisterTenant`, `IsTenantRegistered`, `GetRegisteredTenants` | `db/registered_tenants.go` | Deployed |
| Tenant registration endpoint `POST /api/v1/tenants/register` — open to any authenticated OC Tenant Admin (verified via `oc_tenant_members`); auto-assigns `tenant-admin` to ALL OC Tenant Admins of the tenant | `tenant_handler.go` | Deployed |
| `oc_roles` DB table — stores OC tenant role + service roles (JSONB) per user per tenant (requires users FK) | `db/` | Deployed |
| `oc_tenant_members` DB table — stores ALL Horizon members by email/display_name/oc_role/**member_type**, no FK to users; source for member picker | `db/` | Deployed |
| DB methods — `UpsertOCRoles`, `DeleteOCRoles`, `GetOCRolesForUser`, `UpsertTenantMember`, `DeleteTenantMember`, `GetTenantAdmins`, `GetTenantMembers` | `db/roles.go` | Deployed |
| Login-triggered OC sync in `CallbackHandler` — **synchronous** (no goroutine). Parses JWT for own data; calls `GET /v0/tenants/{rns}` (admins list) + `GET /v0/tenants/{rns}/members` to derive roles; writes ALL members to `oc_tenant_members`; writes UCP users to `oc_roles`; assigns `tenant-admin` for registered tenants | `bff_auth.go` | Deployed |
| `parseOCGroupsFromJWT` — derives tenant list + service roles from JWT `groups` claim | `bff_auth.go` | Deployed |
| `GET /api/v1/me/tenants` — reads from `oc_roles` + `ucp_registered_tenants` + `tenant_role_assignments`; admin contacts from `oc_tenant_members`; zero Horizon calls | `tenant_handler.go` | Deployed |
| Tenant page — 4-state rendering + inline role management; Revoke button hidden for current user's own row | `TenantInfo.jsx`, frontend | Deployed |
| `GET /api/v1/admin/tenants/{slug}/members` — open to all authenticated users; reads from `oc_tenant_members` LEFT JOIN users + `tenant_role_assignments` | `rbac_handler.go` | Deployed |
| ~~Background sync job~~ | — | Deferred — notes only (login sync is sufficient for PoC) |

---

## Scope

### In Scope

**Phase 1 + 2 — Core RBAC + Permission model (deployed)**
- 3-role model (`developer`, `tenant-admin`, `platform-admin`) with `Permission` bitmask
- `RequirePermission` middleware: resolves tenant from `?tenantId=` or `{tenantSlug}`, targeted role fetch (≤2 rows), bitmask check
- `developer` has `PermProvision` but not `PermApprove` — cannot approve own requests
- Admin API for role assignments, slug-based paths
- `/auth/me` extended with roles map and `isPlatformAdmin`
- Permission-aware frontend — sidebar, page buttons, route guards, inline role management in tenant page

**Phase 3 — Tenant onboarding + login-triggered sync**
- **Tenant registration** — `ucp_registered_tenants` table; a tenant must be explicitly registered by an OC Tenant Admin before UCP operations are allowed
- **`oc_roles` table** — populated on every login with each user's OC tenant role and service roles (JSONB). Not used for access control today; wired up for [Option 2](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6645566515/UCP+Identity+Tenancy+Roles) (runtime OC service-role check) when needed
- **Login-triggered OC sync** — runs **synchronously** in the OIDC callback (no goroutine) so all data is ready before the browser redirects. Parses JWT groups for own data; calls `GET /v0/tenants/{rns}` (admins list) + `GET /v0/tenants/{rns}/members` for all members. Writes ALL members to `oc_tenant_members` (no UCP account required). OC role is derived by cross-referencing: email in `admins[]` = "Tenant Admin", else "Tenant Member". UPSERTs `oc_roles` for members with UCP accounts. Assigns `tenant-admin` only for registered tenants. Revokes for removed members.
- **Tenant registration** — open to any authenticated OC Tenant Admin (verified via `oc_tenant_members`, no UCP role required). Auto-assigns `tenant-admin` to ALL OC Tenant Admins of the tenant found in `oc_tenant_members` — not just the registrant.
- **`GET /api/v1/me/tenants`** — returns the user's OC tenants enriched with UCP registration status, UCP role, and tenant-admin contact info (name + email) for tenants where the user has no role or the tenant is unregistered
- **Tenant page** — 4-state rendering per tenant. Member list (`GET /admin/tenants/{slug}/members`) visible to all authenticated users in both State A (has role) and State B (no role) — read-only for non-admins, assign/revoke actions for tenant-admins only. Sources from locally-synced `oc_roles` table — no live Horizon call at view time.

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
data. The member list (`GET /api/v1/admin/tenants/{slug}/members`) is open to
all authenticated users — any tenant member can see who else is in the tenant
and their UCP role. Only tenant-admins see assign/revoke actions. Non-members
cannot be assigned a role through the normal UI flow since only OC-synced
members appear in the list.

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

**Partially resolved.** `parseOCGroupsFromJWT` is deployed and replaces the
`fetchOCMemberTenants` Horizon call for the logged-in user's own data. The
assumption — that Tenant Members have `rns:roc:iam::{tenant}:roles:member` —
is implemented but not yet verified with a non-admin test account.

`GET /v0/tenants/{rns}/members` (Horizon) remains in use for syncing other
members of a tenant. The JWT only encodes the currently authenticated user.

---

### 7. Should OC tenant members share the `users` table or stay separate?

Currently UCP uses two tables for people:

- **`users`** — only people who have logged in to UCP at least once (JIT via OIDC callback). Has `idp_id`, `external_id`, `last_login_at`.
- **`oc_tenant_members`** — ALL Horizon members (email, display_name, oc_role, member_type). No FK to `users`. Populated from the login sync regardless of UCP login status.

The gap this creates: role assignments (`tenant_role_assignments`) have a FK to `users(id)`, so OC members who haven't logged in can't be pre-assigned a UCP role. They get it automatically on their first login, but there's a window where they appear in the member list with no UCP role.

**Option A — Keep separate (current approach)**

| Aspect | Detail |
|---|---|
| Simplicity | Two tables, clear separation between "OC member" and "UCP user" |
| Role assignment | Can only assign to members who have logged in |
| Table size | `users` stays lean — only active UCP users |
| Pre-login gap | OC Tenant Admins get role on first login, not at registration |

**Option B — Merge into `users` with an activity flag**

OC members are pre-provisioned into `users` with `last_login_at = null` (or an `is_active` flag). `tenant_role_assignments` can reference them immediately.

| Aspect | Detail |
|---|---|
| Simplicity | One table, simpler FK model |
| Role assignment | Can pre-assign roles at registration for all OC admins |
| Table size | `users` grows with all OC members even if they never use UCP |
| `idp_id`/`external_id` | Unknown before OIDC — require placeholder values or nullable columns |
| JIT provisioning | Must be updated on first login to fill in Keycloak identity fields |

The PoC uses Option A. Option B would eliminate `oc_tenant_members` and the pre-login gap but requires schema changes to `users` and a two-phase provisioning model.

---

### 8. How should non-user member types be handled?

The OC member list includes a `type` field on each member (`"user"`,
`"service_account"`, `"team"`, etc.). UCP stores this in `oc_tenant_members.member_type`
and surfaces it in `GET /admin/tenants/{slug}/members` as `memberType`.

Open questions:
- Should service accounts and teams be **excluded from the member picker**? A
  `service_account` cannot log in via OIDC, so UCP role assignment is meaningless
  for them. A `team` represents a group — role semantics are unclear.
- Should UCP role assignment be **blocked** for non-user members (backend
  validation in `AssignRole`)?
- What are the exact `type` values returned by Horizon? The admins array from
  `GET /v0/tenants/{rns}` shows `"type": "user"` — other values are unverified.

The PoC stores and surfaces `memberType` but does not yet filter or restrict
based on it. The frontend currently shows all member types in the role management UI.

---

### 8. Can a tenant-admin demote another OC Tenant Admin?

A UCP `tenant-admin` can revoke another user's UCP role via `DELETE /admin/tenants/{slug}/roles/{userID}`. However, if the target user is an OC Tenant Admin and the tenant is registered, the login sync will re-assign them `tenant-admin` on their next login.

This means:
- Revoking a UCP role from an OC Tenant Admin is **transient** — it is restored on next login
- There is no mechanism in the PoC to permanently suppress a UCP role that the sync would re-grant

Open questions:
- Should `RevokeRole` block revocation of other OC Tenant Admins (since it would be restored anyway)?
- Or should revocation be permanent and the sync respect a "manually revoked" flag?
- Should a `tenant-admin` be able to demote another `tenant-admin` at all, or is that a `platform-admin`-only operation?

The PoC does not restrict this — a `tenant-admin` can revoke any other user's role (except their own), but the sync will restore OC Tenant Admins on next login.

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
| `/api/v1/tenants/{tenantSlug}/workflows/{workflowId}/approve` | POST | `tenant-admin` | Approve a pending Temporal workflow |
| `/api/v1/tenants/{tenantSlug}/workflows/{workflowId}/reject` | POST | `tenant-admin` | Reject a pending Temporal workflow |
| `/api/v1/drift` | GET | `tenant-admin` | List drift detections for the tenant |
| `/api/v1/quota` | GET | `tenant-admin` | View GCP quota usage for the tenant |
| `/api/v1/gcp/discover` | GET | `tenant-admin` | Discover unmanaged GCP resources for the tenant |
| `/api/v1/settings/api-exposure` | GET, PUT | `tenant-admin` | Read or configure API exposure settings |
| `/api/v1/settings/credentials` | GET, POST | `tenant-admin` | Read or upload GCP credentials for the tenant |
| `/api/v1/settings/credentials/{provider}` | DELETE | `tenant-admin` | Remove a provider's credentials for the tenant |
| `/api/v1/settings/credentials/roc` | GET, POST | `tenant-admin` | Read or configure ROC (Omnia) credentials for the tenant |
| `/api/v1/settings/credentials/all` | GET | `platform-admin` | List all credentials across all tenants |
| `/api/v1/admin/tenants/{tenantSlug}/members` | GET | — (any authenticated user) | List OC-synced members with their UCP role |
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
| Database → List | `/` | `developer` | Hidden + 403 |
| Database → Create | `/create` | `developer` | Hidden in sidebar + 403 on direct access |
| Compute → List | `/compute` | `developer` | Hidden + 403 |
| Compute → Create | `/compute/create` | `developer` | Hidden + 403 |
| Storage → List | `/storage` | `developer` | Hidden + 403 |
| Storage → Create | `/storage/create` | `developer` | Hidden + 403 |
| Kubernetes → List | `/kubernetes` | `developer` | Hidden + 403 |
| Kubernetes → Create | `/kubernetes/create` | `developer` | Hidden + 403 |
| Drift → List | `/drift` | `tenant-admin` | Hidden + 403 |
| Quotas → Usage | `/quota` | `tenant-admin` | Hidden + 403 |
| Terraform | `/terraform/*` | — | Out of RBAC scope (experimental) |
| Admin → Workflows | `/admin/workflows` | `platform-admin` | Hidden + 403 |
| Admin → Kube Resources | `/admin/kube-resources` | `platform-admin` | Hidden + 403 |
| Settings → Credentials | `/settings/credentials` | `tenant-admin` | Hidden + 403 |
| Tenants | `/admin/tenants` | — | Accessible to all authenticated users |

### In-page actions

| Action | Page | Minimum role |
|---|---|---|
| Create button | All resource list pages | `developer` |
| Delete button | All resource list/detail pages | `developer` |
| Approve button | Workflows page | `tenant-admin` |
| Reject button | Workflows page | `tenant-admin` |
| Upload credentials | Settings → Credentials | `tenant-admin` |
| View member list | Tenant page (inline) | — (any authenticated user) |
| Assign / revoke role | Tenant page (inline) | `tenant-admin` |

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
