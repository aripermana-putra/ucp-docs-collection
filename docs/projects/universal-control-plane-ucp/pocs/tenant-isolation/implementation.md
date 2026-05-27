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
| `xrBelongsToTenant()` | `main.go` | Deployed |
| `xrBelongsToAnyTenant()` | `main.go` | Deployed |
| `getUserTenantIDs()` | `horizon_handler.go` | Deployed |
| `errTenantNotFound` / `isTenantNotFound()` | `settings_handler.go` | Deployed |
| Label + annotation stamping — XDatabase | `main.go` (`createXDatabaseYAML`) | Deployed |
| Label + annotation stamping — XKubernetesCluster | `kubernetes_handler.go` (`createXKubernetesClusterYAML`) | Deployed |
| Label + annotation stamping — XComputeInstance, XStorageBucket, XLoadBalancer | `compute_handler.go`, `storage_handler.go`, `load_balancer_handler.go` | Deployed |
| Tenant-filtered list — `?tenantId=` query param or caller's tenants via Horizon | `main.go`, `kubernetes_handler.go`, `compute_handler.go`, `storage_handler.go`, `load_balancer_handler.go` | Deployed |
| Delete ownership check — all resource types | `main.go`, `kubernetes_handler.go`, `compute_handler.go`, `storage_handler.go`, `load_balancer_handler.go` | Deployed |
| Global `TenantContext` + `?tenantId=` query param injection on GET requests | `TenantContext.jsx`, `useAuthFetch.ts`, `MainLayout.jsx`, frontend | Deployed |
| `ValidatingAdmissionPolicy` — tenant label + annotation enforcement | `k8s/tenant-admission-policy.yaml` | Deployed |
| Tenant slug resolver — `ResolveTenantExternalIDBySlug` | `db/tenants.go` | Not deployed |
| Slug-based mutation paths — `DELETE/POST /api/v1/tenants/{tenantSlug}/...` | all resource handlers + `main.go` | Not deployed |
| Tenant-slug URL construction for mutations | `useAuthFetch.ts`, frontend list components | Not deployed |
| `tenantId` field in list response objects | all resource response structs | Not deployed |

---

## Scope

### In Scope

- **Label stamping** — every XR creation stamps `platform.ucp.io/tenant` (label) and
  `platform.ucp.io/tenant-id` (annotation) on the Kubernetes object
- **Tenant-filtered list endpoints** — optional `?tenantId=` query param scopes
  results to a specific tenant (caller must be admin); when absent, results are scoped
  to all tenants the caller belongs to via Horizon membership lookup
- **Slug-based mutation paths** — DELETE, approve, and reject operations include the
  tenant slug as the first path segment (`/api/v1/tenants/{tenantSlug}/...`); the API
  resolves slug → canonical RNS via the `tenants` table before the ownership check
- **Delete ownership enforcement** — delete endpoints read the XR's
  `platform.ucp.io/tenant-id` annotation and reject callers who are not admin of that
  tenant (HTTP 403); 422 if the annotated tenant does not exist in Horizon
- **Tenant ID in list responses** — each resource in a list response includes a
  `tenantId` field so callers can identify which tenant owns each resource
- **ProviderConfig injection** — `providerConfig` is always derived server-side from
  `gcpProviderConfigName(tenantID, env)`; callers cannot supply it
- **ValidatingAdmissionPolicy** — Kubernetes admission webhook that rejects XR creation
  if `platform.ucp.io/tenant` label or `platform.ucp.io/tenant-id` annotation is absent

### Out of Scope

- Terraform endpoints (`/api/v1/terraform/*`) — these use the OpenTofu `random` provider
  with no cloud credentials; per-tenant ProviderConfig is not applicable
- Omnia-specific isolation — handled at the auth layer via per-tenant JWK + on-demand JWT

---

## API Sequence — Create with Admission Validation

`POST /api/v1/storage-buckets` (same flow applies to all resource types)

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Handler as API Server<br/>(CreateStorageBucket)
    participant Horizon as Horizon API
    participant Temporal as Temporal Worker
    participant K8sAPI as Kubernetes API Server<br/>(ValidatingAdmissionPolicy)
    participant Crossplane

    Browser->>Handler: POST /api/v1/storage-buckets
    Note over Browser,Handler: Cookie: session=abc123<br/>X-Environment: QA<br/>Body: {name, projectId, tenantId: "rns:roc:iam::clsd-ucp"}

    Handler->>Horizon: GET /v0/tenants/rns:roc:iam::clsd-ucp
    Horizon-->>Handler: {admins: [...]}
    Handler->>Handler: caller ∈ admins → authorized

    Handler->>Handler: stamp XR metadata<br/>label  platform.ucp.io/tenant: "rns-roc-iam--clsd-ucp"<br/>annot  platform.ucp.io/tenant-id: "rns:roc:iam::clsd-ucp"<br/>param  providerConfig: "gcp-rns-roc-iam--clsd-ucp-qa"

    Handler->>Temporal: ExecuteWorkflow("RequestStorageWorkflow", xrYAML)
    Handler-->>Browser: 202 Accepted {workflowId}

    Temporal->>K8sAPI: CREATE XObjectStorage (xrYAML)

    K8sAPI->>K8sAPI: evaluate ValidatingAdmissionPolicy CEL<br/>① has(object.metadata.labels) && 'platform.ucp.io/tenant' in labels?<br/>② has(object.metadata.annotations) && 'platform.ucp.io/tenant-id' in annotations?

    alt both expressions true
        K8sAPI-->>Temporal: 201 Created
        K8sAPI--)Crossplane: watch event — new XR
        Crossplane->>Crossplane: reconcile → provision resource
    else label or annotation absent
        K8sAPI-->>Temporal: 422 Unprocessable Entity<br/>"XR must carry the platform.ucp.io/tenant label"
        Note over Temporal: workflow fails — XR never persisted
    end
```

The same policy covers `UPDATE` operations. A direct `kubectl patch` on an existing XR
that removes the label or annotation is also rejected at the API server layer before
reaching etcd.

---

## API Sequence — Tenant-Filtered List

`GET /api/v1/databases?tenantId=rns:roc:iam::clsd-ucp`

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Handler as API Server<br/>(ListDatabases)
    participant Horizon as Horizon API
    participant K8s as Kubernetes

    Browser->>Handler: GET /api/v1/databases?tenantId=rns:roc:iam::clsd-ucp
    Note over Browser,Handler: Cookie: session=abc123<br/>X-Environment: QA

    Handler->>Handler: tenantID = r.URL.Query().Get("tenantId")<br/>= "rns:roc:iam::clsd-ucp"

    Handler->>Horizon: GET /v0/tenants/rns:roc:iam::clsd-ucp
    Horizon-->>Handler: {"admins": [{"email": "user@example.com"}]}
    Handler->>Handler: email ∈ admins → authorized

    Handler->>K8s: LIST xdatabases.platform.example.io (no server-side filter)
    K8s-->>Handler: all XDatabase objects

    loop for each XR
        Handler->>Handler: xrBelongsToTenant(item, tenantID, env)<br/>1. annotation platform.ucp.io/tenant-id == tenantID?<br/>2. label platform.ucp.io/tenant == sanitizeTenantID(tenantID)?<br/>3. spec.parameters.providerConfig == gcpProviderConfigName(tenantID, env)?
        Note over Handler: skip if none match
    end

    Handler-->>Browser: 200 OK {"databases": [{..., "tenantId": "rns:roc:iam::clsd-ucp"}, ...]}
```

When `?tenantId=` is absent the handler calls `getUserTenantIDs()` via Horizon to
fetch all tenants the caller belongs to, and filters across all of them.

---

## API Sequence — Delete with Ownership Check

`DELETE /api/v1/tenants/{tenantSlug}/databases/{name}`

The tenant slug is a URL-safe short name (e.g. `clsd-ucp`). The handler resolves
it to the canonical RNS before performing the ownership check.

```mermaid
sequenceDiagram
    autonumber

    participant Browser
    participant Handler as API Server<br/>(DeleteDatabase)
    participant DB as PostgreSQL
    participant Horizon as Horizon API
    participant K8s as Kubernetes

    Browser->>Handler: DELETE /api/v1/tenants/clsd-ucp/databases/my-db

    Handler->>DB: ResolveTenantExternalIDBySlug("clsd-ucp")
    DB-->>Handler: "rns:roc:iam::clsd-ucp"

    Handler->>K8s: GET xdatabase/my-db
    K8s-->>Handler: XDatabase object

    Handler->>Handler: xrTenantID = annotations["platform.ucp.io/tenant-id"]<br/>fallback: spec.parameters.tenantId

    alt xrTenantID is set
        Handler->>Horizon: GET /v0/tenants/{xrTenantID}
        Horizon-->>Handler: admins list OR 404

        alt Horizon returns 404 (unknown tenant)
            Handler-->>Browser: 422 Unprocessable Entity<br/>"Resource belongs to an unknown tenant"
        else caller not in admins
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

# Own tenant → 200, filtered results (tenantId as query param)
curl -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/databases?tenantId=rns:roc:iam::clsd-ucp"

# Tenant the caller is not admin of → 403
curl -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/databases?tenantId=rns:roc:iam::other-tenant"

# No tenantId → 200, all caller's tenants' resources
curl -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/databases"
```

### Delete ownership

```bash
# Stamp a foreign tenant on an existing XR
kubectl annotate xdatabase <name> \
  platform.ucp.io/tenant-id=rns:roc:iam::other-tenant --overwrite

# Delete attempt using slug path — 403 (caller not admin of other-tenant)
curl -X DELETE -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/tenants/clsd-ucp/databases/<name>"
```

---

## Limitations

### List handlers scope to caller's tenants when X-Tenant-ID is absent

When `X-Tenant-ID` is not set, list handlers call Horizon
`GET /v0/members/<email>/tenants` to fetch all tenants the caller belongs to and
filter results to those tenants. This means a user who is a member of multiple
tenants sees resources across all of them in the default (no header) view.

### providerConfig fallback relies on naming convention

The third fallback in `xrBelongsToTenant` matches by ProviderConfig name. This relies on
the naming convention `gcp-{sanitized-tenant-id}-{env}` being stable. XRs provisioned with
a non-standard ProviderConfig name are not matched by any fallback.

### Delete check is skipped when annotation is absent

If an XR carries neither the `platform.ucp.io/tenant-id` annotation nor
`spec.parameters.tenantId`, the delete ownership check is skipped and the deletion
proceeds. This applies to XRs created before label stamping was deployed.
