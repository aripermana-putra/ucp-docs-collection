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

`resolveUserRole` lives in `rbac_handler.go`. It calls the Horizon members
endpoint and maps the returned `default_role` to the internal `Role` type.

```go
// rbac_handler.go
func (s *APIServer) resolveUserRole(r *http.Request, tenantID string) (Role, error) {
    authToken := extractBearerToken(r)
    principal, _ := authpkg.PrincipalFromContext(r.Context())

    env := r.Header.Get("X-Environment")
    if env == "" {
        env = "PRODUCTION"
    }

    endpoint := fmt.Sprintf("%s/members/%s/tenants?subscriptions=true",
        getHorizonAPIBaseURL(env),
        url.PathEscape(principal.Email),
    )

    body, status, err := s.horizonGet(authToken, endpoint)
    if err != nil || status != http.StatusOK {
        return RoleUnknown, fmt.Errorf("horizon returned %d: %w", status, err)
    }

    var resp struct {
        Items []struct {
            RNS           string `json:"rns"`
            Subscriptions []struct {
                DefaultRole string `json:"default_role"`
                Service     string `json:"service"`
            } `json:"subscriptions"`
        } `json:"items"`
    }
    if err := json.Unmarshal(body, &resp); err != nil {
        return RoleUnknown, fmt.Errorf("failed to parse tenant list: %w", err)
    }

    for _, item := range resp.Items {
        // platform-admin: present on any tenant entry
        for _, sub := range item.Subscriptions {
            if sub.DefaultRole == "platform-admin" {
                return RolePlatformAdmin, nil
            }
        }
        // tenant-scoped role
        if item.RNS == tenantID {
            for _, sub := range item.Subscriptions {
                return horizonRoleToInternal(sub.DefaultRole), nil
            }
        }
    }
    return RoleUnknown, nil
}

func horizonRoleToInternal(r string) Role {
    switch r {
    case "tenant-admin": return RoleTenantAdmin
    case "deployer":     return RoleDeployer
    case "approver":     return RoleApprover
    case "viewer":       return RoleViewer
    default:             return RoleUnknown
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

## open question — platform-admin Resolution

The Horizon API `GET /v0/members/<email>/tenants?subscriptions=true` returns
a list of tenants the caller belongs to. A `platform-admin` is identified when
any entry in that list carries `default_role: platform-admin`.

This assumes Horizon uses `platform-admin` as a literal `default_role` value.
If Horizon uses a different string (e.g. `admin` or a group name), the mapping
in `horizonRoleToInternal` must be updated to match the actual value returned.
The exact value should be confirmed against a live Horizon response during
implementation.
