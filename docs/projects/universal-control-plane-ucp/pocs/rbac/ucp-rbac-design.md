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

## resolveUserRole

`resolveUserRole` lives in `rbac_handler.go`. By the time it is called,
`SessionMiddleware` has already run and injected a `Principal` into the request
context. The `Principal` provides what both candidate approaches need without
any additional network call for the role lookup itself:

```go
principal, _ := authpkg.PrincipalFromContext(r.Context())
// principal.AccessToken — decrypted Keycloak JWT (Option B)
// principal.UserID      — internal DB UUID       (Option C)
```

### Option B — Keycloak JWT claims

Requires `UCPClaims` in `auth/auth.go` to carry a custom `ucp_roles` claim
configured in Keycloak via a protocol mapper:

```go
type UCPClaims struct {
    jwt.RegisteredClaims
    Email             string            `json:"email"`
    PreferredUsername string            `json:"preferred_username"`
    Name              string            `json:"name"`
    UCPRoles          map[string]string `json:"ucp_roles"`
}

func (s *APIServer) resolveUserRole(r *http.Request, tenantID string) (Role, error) {
    principal, _ := authpkg.PrincipalFromContext(r.Context())
    claims, err := authpkg.ParseUnverified(principal.AccessToken)
    if err != nil {
        return RoleUnknown, fmt.Errorf("failed to parse token claims: %w", err)
    }
    roleStr, ok := claims.UCPRoles[tenantID]
    if !ok {
        // check platform-admin (not tenant-scoped)
        if claims.UCPRoles["*"] == "platform-admin" {
            return RolePlatformAdmin, nil
        }
        return RoleUnknown, nil
    }
    return stringToRole(roleStr), nil
}
```

### Option C — UCP database

Requires a `tenant_role_assignments` table and a `GetTenantRole` DB method:

```go
func (s *APIServer) resolveUserRole(r *http.Request, tenantID string) (Role, error) {
    principal, _ := authpkg.PrincipalFromContext(r.Context())
    roleStr, err := s.db.GetTenantRole(principal.UserID, tenantID)
    if err != nil {
        return RoleUnknown, fmt.Errorf("failed to resolve role: %w", err)
    }
    return stringToRole(roleStr), nil
}
```

### Shared mapping

```go
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

## Undecided — Role Resolution Approach

The role resolution mechanism has not been decided. Two candidates:

| | Option B — Keycloak JWT | Option C — UCP Database |
|---|---|---|
| Source | `ucp_roles` claim in access token | `tenant_role_assignments` table |
| Extra call at resolution | None — token already in `Principal` | One DB query |
| Role change latency | Token TTL (~10 min) | Immediate |
| Setup | Keycloak protocol mapper + group assignment | Schema migration + admin API |
| `platform-admin` representation | `UCPRoles["*"] = "platform-admin"` | Row with `tenant_rns = '*'` |

See `concepts.md` for detailed description of each option.
