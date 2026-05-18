---
title: "Tenant Isolation — Implementation"
space: UCP
parent_page_id: "../tenant-isolation.md"
---

# Tenant Isolation — Implementation

The API server enforces tenant isolation through three mechanisms: ownership label
stamping at XR creation, tenant-filtered list endpoints, and ownership verification
before delete operations.

---

## Implementation

| Component | File | Status |
|---|---|---|
| `sanitizeTenantID()` | `settings_handler.go:290` | Deployed |
| `gcpProviderConfigName()` | `settings_handler.go:236` | Deployed |
| `isUserTenantAdmin()` | `horizon_handler.go` | Deployed |
| `xrBelongsToTenant()` | `main.go:1308` | Deployed |
| Label + annotation stamping — XDatabase | `main.go` (`createXDatabaseYAML`) | Deployed |
| Label + annotation stamping — XKubernetesCluster | `kubernetes_handler.go` (`createXKubernetesClusterYAML`) | Deployed |
| Label + annotation stamping — XComputeInstance, XStorageBucket, XLoadBalancer | `compute_handler.go`, `storage_handler.go`, `load_balancer_handler.go` | Deployed |
| Tenant-filtered `ListDatabases` | `main.go` | Deployed |
| Tenant-filtered `ListKubernetesClusters` | `kubernetes_handler.go` | Deployed |
| Tenant-filtered list — compute, storage, load balancer | `compute_handler.go`, `storage_handler.go`, `load_balancer_handler.go` | Deployed |
| Delete ownership check — XDatabase | `main.go` (`DeleteDatabase`) | Deployed |
| Delete ownership check — XKubernetesCluster | `kubernetes_handler.go` (`DeleteKubernetesCluster`) | Deployed |
| GCP credential upload + ProviderConfig auto-creation | `settings_handler.go` (`upsertGCPProviderConfig`) | Deployed |
| Per-tenant Ed25519 JWK + on-demand JWT for Omnia | `settings_handler.go` | Deployed |
| Global `TenantContext` + `X-Tenant-ID` header injection | `useAuthFetch.ts`, frontend | Not deployed |
| `X-Tenant-ID` fallback in list handlers | `main.go`, `kubernetes_handler.go`, `compute_handler.go` | Not deployed |
| `ValidatingAdmissionPolicy` — tenant label + annotation enforcement | `k8s/` (new manifest) | Not deployed |

---

## Scope

### In Scope

- **Label stamping** — every XR creation stamps `platform.ucp.io/tenant` (label) and
  `platform.ucp.io/tenant-id` (annotation) on the Kubernetes object
- **Tenant-filtered list endpoints** — `?tenantId=<rns>` query parameter activates
  tenant filtering; the caller must be admin of the specified tenant
- **Delete ownership enforcement** — delete endpoints read the XR's
  `platform.ucp.io/tenant-id` annotation and reject callers who are not admin of that
  tenant (HTTP 403)
- **ProviderConfig injection** — `providerConfig` is always derived server-side from
  `gcpProviderConfigName(tenantID, env)`; callers cannot supply it
- **ValidatingAdmissionPolicy** — Kubernetes admission webhook that rejects XR creation
  if `platform.ucp.io/tenant` label or `platform.ucp.io/tenant-id` annotation is absent

### Out of Scope

- Terraform endpoints (`/api/v1/terraform/*`) — these use the OpenTofu `random` provider
  with no cloud credentials; per-tenant ProviderConfig is not applicable
- Omnia-specific isolation — handled at the auth layer via per-tenant JWK + on-demand JWT

---

## API Sequence — Tenant-Filtered List

`GET /api/v1/databases?tenantId=rns:roc:iam::clsd-ucp`

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant SessionDB as PostgreSQL<br/>(session store)
    participant Handler as API Server<br/>(ListDatabases)
    participant Horizon as Horizon API
    participant K8s as Kubernetes

    Browser->>Handler: GET /api/v1/databases?tenantId=rns:roc:iam::clsd-ucp
    Note over Browser,Handler: Cookie: session=abc123<br/>X-Environment: QA

    Handler->>SessionDB: GetSessionWithUser("abc123")
    SessionDB-->>Handler: session{enc_access_token, ...}
    Note over Handler: decrypt → Principal{AccessToken} injected into context

    Handler->>Handler: tenantID = "rns:roc:iam::clsd-ucp"

    Handler->>Horizon: GET /v0/tenants/rns:roc:iam::clsd-ucp
    Note over Handler,Horizon: Authorization: Bearer {access_token}
    Horizon-->>Handler: {"admins": [{"email": "user@example.com"}]}
    Handler->>Handler: email ∈ admins → authorized

    Handler->>K8s: LIST xdatabases.platform.example.io (no server-side filter)
    K8s-->>Handler: all XDatabase objects

    loop for each XR
        Handler->>Handler: xrBelongsToTenant(item, tenantID, env)<br/>1. annotation platform.ucp.io/tenant-id == tenantID?<br/>2. label platform.ucp.io/tenant == sanitizeTenantID(tenantID)?<br/>3. spec.parameters.providerConfig == gcpProviderConfigName(tenantID, env)?
        Note over Handler: skip if none match
    end

    Handler-->>Browser: 200 OK {"databases": [...filtered...]}
```

---

## API Sequence — Delete with Ownership Check

`DELETE /api/v1/databases/{name}`

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Handler as API Server<br/>(DeleteDatabase)
    participant Horizon as Horizon API
    participant K8s as Kubernetes

    Browser->>Handler: DELETE /api/v1/databases/my-db

    Handler->>K8s: GET xdatabase/my-db
    K8s-->>Handler: XDatabase object

    Handler->>Handler: xrTenantID = annotations["platform.ucp.io/tenant-id"]<br/>fallback: spec.parameters.tenantId

    alt xrTenantID is set
        Handler->>Horizon: GET /v0/tenants/{xrTenantID}
        Horizon-->>Handler: admins list
        Handler->>Handler: caller ∈ admins?

        alt not admin
            Handler-->>Browser: 403 Forbidden
        end
    end

    Handler->>K8s: DELETE xdatabase/my-db
    K8s-->>Handler: 200 OK
    Handler-->>Browser: 200 OK {"message": "deleted"}
```

---

## Key Functions

### gcpProviderConfigName

```go
// settings_handler.go:236
func gcpProviderConfigName(tenantID, env string) string {
    e := "production"
    if strings.ToLower(env) == "qa" {
        e = "qa"
    }
    return fmt.Sprintf("gcp-%s-%s", sanitizeTenantID(tenantID), e)
}
```

Produces: `gcp-rns-roc-iam--clsd-ucp-qa`

### xrBelongsToTenant

```go
// main.go:1308
func xrBelongsToTenant(obj *unstructured.Unstructured, tenantID, env string) bool {
    if obj.GetAnnotations()["platform.ucp.io/tenant-id"] == tenantID {
        return true
    }
    if obj.GetLabels()["platform.ucp.io/tenant"] == sanitizeTenantID(tenantID) {
        return true
    }
    providerConfig, _, _ := unstructured.NestedString(obj.Object, "spec", "parameters", "providerConfig")
    return providerConfig == gcpProviderConfigName(tenantID, env)
}
```

---

## Verification

### Label stamping

Create a resource via the UI, then inspect the XR:

```bash
kubectl get xdatabase <name> -o jsonpath='{.metadata.labels}' | jq .
# {"platform.ucp.io/tenant": "rns-roc-iam--clsd-ucp"}

kubectl get xdatabase <name> -o jsonpath='{.metadata.annotations}' | jq .
# {"platform.ucp.io/tenant-id": "rns:roc:iam::clsd-ucp", ...}
```

### List filtering

```bash
COOKIE="Cookie: session=<value-from-browser-devtools>"

# Caller's own tenant → 200, filtered results
curl -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/databases?tenantId=rns:roc:iam::clsd-ucp"

# Tenant the caller is not admin of → 403
curl -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/databases?tenantId=rns:roc:iam::other-tenant"
```

### Delete ownership

```bash
# Stamp a foreign tenant on an existing XR
kubectl annotate xdatabase <name> \
  platform.ucp.io/tenant-id=rns:roc:iam::other-tenant --overwrite
kubectl label xdatabase <name> \
  platform.ucp.io/tenant=rns-roc-iam--other-tenant --overwrite
kubectl patch xdatabase <name> --type=merge \
  -p '{"spec":{"parameters":{"providerConfig":"gcp-rns-roc-iam--other-tenant-qa"}}}'

# Delete attempt → 403
curl -X DELETE -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/databases/<name>"
```

---

## Limitations

### List endpoints require explicit tenantId

List handlers return all resources when `?tenantId=` is absent. The browser UI does not
currently send a tenant ID on list requests — the frontend has no global tenant context.
The design fix (global `TenantContext` + `X-Tenant-ID` header) is described in
`ucp-tenant-isolation-design.md`.

### providerConfig fallback relies on naming convention

The third fallback in `xrBelongsToTenant` matches by ProviderConfig name. This relies on
the naming convention `gcp-{sanitized-tenant-id}-{env}` being stable. XRs provisioned with
a non-standard ProviderConfig name are not matched by any fallback.

### Delete check is skipped when annotation is absent

If an XR carries neither the `platform.ucp.io/tenant-id` annotation nor
`spec.parameters.tenantId`, the delete ownership check is skipped and the deletion
proceeds. This applies to XRs created before label stamping was deployed.
