---
title: "RBAC — UCP Design"
space: UCP
parent_page_id: "../rbac.md"
---

# RBAC — UCP Design

---

## Role Type

`Role` is added to `auth/context.go` alongside the existing `Principal` type.
Using an ordered integer enables minimum-role checks as a single `>=` comparison
rather than a lookup table.

```go
// auth/context.go
type Role int

const (
    RoleUnknown     Role = iota // 0 — unresolved / not a tenant member
    RoleViewer                  // 1
    RoleApprover                // 2
    RoleDeployer                // 3
    RoleTenantAdmin             // 4
    RolePlatformAdmin           // 5
)

func (r Role) String() string {
    switch r {
    case RoleViewer:      return "viewer"
    case RoleApprover:    return "approver"
    case RoleDeployer:    return "deployer"
    case RoleTenantAdmin: return "tenant-admin"
    case RolePlatformAdmin: return "platform-admin"
    default:              return "unknown"
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

## Database Schema

```sql
CREATE TABLE tenant_role_assignments (
    user_id    TEXT NOT NULL REFERENCES users(id),
    tenant_rns TEXT NOT NULL,
    role       TEXT NOT NULL CHECK (role IN (
                   'platform-admin', 'tenant-admin',
                   'deployer', 'approver', 'viewer')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, tenant_rns)
);
```

`platform-admin` is stored as a row with `tenant_rns = '*'` — not tied to any
specific tenant, matches regardless of the `X-Tenant-ID` in the request.

---

## resolveUserRole

`resolveUserRole` lives in `rbac_handler.go`. `SessionMiddleware` has already
run by the time it is called — `Principal.UserID` is available in context:

```go
func (s *APIServer) resolveUserRole(r *http.Request, tenantID string) (Role, error) {
    principal, _ := authpkg.PrincipalFromContext(r.Context())

    // check platform-admin first (tenant_rns = '*')
    if roleStr, err := s.db.GetTenantRole(principal.UserID, "*"); err == nil && roleStr != "" {
        return stringToRole(roleStr), nil
    }

    roleStr, err := s.db.GetTenantRole(principal.UserID, tenantID)
    if err != nil {
        return RoleUnknown, fmt.Errorf("failed to resolve role: %w", err)
    }
    return stringToRole(roleStr), nil
}

func stringToRole(s string) Role {
    switch s {
    case "platform-admin": return RolePlatformAdmin
    case "tenant-admin":   return RoleTenantAdmin
    case "deployer":       return RoleDeployer
    case "approver":       return RoleApprover
    case "viewer":         return RoleViewer
    default:               return RoleUnknown
    }
}
```

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
{ "userEmail": "user@example.com", "role": "deployer" }
```

The handler resolves `userEmail` to a `user_id` via the `users` table (user
must have logged in at least once for JIT provisioning to have created their
record) before inserting into `tenant_role_assignments`.

---

## RequireRole Middleware

`RequireRole` returns a gorilla/mux-compatible middleware. It resolves the
caller's role for the tenant in `X-Tenant-ID`, enforces the minimum, and
injects the resolved role into the context for downstream handlers.

```go
// rbac_handler.go
func (s *APIServer) RequireRole(minRole authpkg.Role) mux.MiddlewareFunc {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            tenantID := strings.TrimSpace(r.Header.Get("X-Tenant-ID"))

            role, err := s.resolveUserRole(r, tenantID)
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

            next.ServeHTTP(w, r.WithContext(authpkg.WithRole(r.Context(), role)))
        })
    }
}
```

---

## Route Grouping

Routes are grouped by minimum required role. Each group gets a dedicated
subrouter with the appropriate middleware applied once — replacing the ad-hoc
`isUserTenantAdmin()` calls in individual handlers.

```go
// main.go — route registration
viewerRoutes := api.PathPrefix("").Subrouter()
viewerRoutes.Use(server.RequireRole(authpkg.RoleViewer))

deployerRoutes := api.PathPrefix("").Subrouter()
deployerRoutes.Use(server.RequireRole(authpkg.RoleDeployer))

approverRoutes := api.PathPrefix("").Subrouter()
approverRoutes.Use(server.RequireRole(authpkg.RoleApprover))

tenantAdminRoutes := api.PathPrefix("").Subrouter()
tenantAdminRoutes.Use(server.RequireRole(authpkg.RoleTenantAdmin))

platformAdminRoutes := api.PathPrefix("").Subrouter()
platformAdminRoutes.Use(server.RequireRole(authpkg.RolePlatformAdmin))

// Resource reads
viewerRoutes.HandleFunc("/databases", server.ListDatabases).Methods("GET")
viewerRoutes.HandleFunc("/databases/{name}", server.GetDatabase).Methods("GET")
// ... other GET endpoints

// Resource mutations
deployerRoutes.HandleFunc("/databases", server.CreateDatabase).Methods("POST")
deployerRoutes.HandleFunc("/databases/{name}", server.DeleteDatabase).Methods("DELETE")
// ... other POST/DELETE endpoints

// Workflow approval
approverRoutes.HandleFunc("/workflows/{workflowId}/approve", server.ApproveWorkflow).Methods("POST")
approverRoutes.HandleFunc("/workflows/{workflowId}/reject", server.RejectWorkflow).Methods("POST")

// Credential management
tenantAdminRoutes.HandleFunc("/settings/credentials", server.GetCredentials).Methods("GET")
tenantAdminRoutes.HandleFunc("/settings/credentials", server.UpsertCredentials).Methods("POST")
tenantAdminRoutes.HandleFunc("/settings/credentials/{provider}", server.DeleteCredentials).Methods("DELETE")

// Platform-wide operations
platformAdminRoutes.HandleFunc("/settings/credentials/all", server.ListAllCredentials).Methods("GET")
```

---

## Handler Cleanup

Once `RequireRole` enforces access at the route level, individual handlers
remove their `isUserTenantAdmin()` calls for create operations. The role check
at the middleware layer is the single enforcement point.

Delete handlers retain `isUserTenantAdmin()` for the **ownership check** — this
reads the tenant from the XR annotation (not from `X-Tenant-ID`) and is a
distinct check that verifies the resource belongs to the caller's tenant. It is
not replaced by `RequireRole`.

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
