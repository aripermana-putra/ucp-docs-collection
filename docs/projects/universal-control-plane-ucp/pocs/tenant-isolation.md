---
title: "Tenant Isolation"
space: UCP
parent_page_id: "../pocs.md"
---

# Tenant Isolation PoC

## Background

MCUCP-119 attempted to migrate all Crossplane XRDs, XRs, and MRs from `scope: Cluster`
to `scope: Namespaced`, using one Kubernetes namespace per tenant as the isolation
boundary. This was halted because Crossplane v2 enforces a hard constraint:

> A namespace-scoped XR can only compose resources in its own namespace.

All managed resources (MRs) from the providers we need are still cluster-scoped:

| Provider | MR Scope | Namespaced Support |
|---|---|---|
| provider-upjet-gcp | Cluster | Not yet (upstream in progress) |
| provider-upjet-aws | Namespace | Done in v2 |
| provider-upjet-azure | Cluster | Not yet (upstream in progress) |
| provider-roc (Omnia) | Cluster | Not yet (we own this — unblocked) |
| provider-terraform | Cluster | Not yet |

Since namespace-per-tenant is blocked upstream, the chosen interim strategy is
**ProviderConfig-per-tenant with API-layer label filtering**.

---

## Strategy: ProviderConfig Per Tenant + API-Layer Filtering

Two complementary mechanisms enforce isolation:

### 1. ProviderConfig Per Tenant (Cloud-Level)

Each tenant gets one `ProviderConfig` per provider, pointing to their dedicated cloud
account:

```
tenant rns:roc:iam::clsd-ucp ->
  ProviderConfig: gcp-rns-roc-iam--clsd-ucp-qa         -> GCP project (QA)
  ProviderConfig: gcp-rns-roc-iam--clsd-ucp-production -> GCP project (production)
```

When the API server creates an XR for a tenant, it injects the correct ProviderConfig
server-side. Tenants never interact with Kubernetes directly — they cannot reference
another tenant's ProviderConfig.

The naming convention is deterministic:

```go
// settings_handler.go
func gcpProviderConfigName(tenantID, env string) string {
    // e.g. "gcp-rns-roc-iam--clsd-ucp-qa"
    return fmt.Sprintf("gcp-%s-%s", sanitizeTenantID(tenantID), env)
}

func sanitizeTenantID(tenantID string) string {
    // "rns:roc:iam::clsd-ucp" -> "rns-roc-iam--clsd-ucp"
    return strings.NewReplacer(":", "-").Replace(strings.ToLower(tenantID))
}
```

### 2. Tenant Ownership Labels on XRs (API-Layer)

Every XR created via the API server carries two ownership markers:

| Type | Key | Value | Purpose |
|---|---|---|---|
| Label | `platform.ucp.io/tenant` | `rns-roc-iam--clsd-ucp` (sanitized) | Kubernetes label selectors — `:` not allowed in label values |
| Annotation | `platform.ucp.io/tenant-id` | `rns:roc:iam::clsd-ucp` (raw) | Ownership checks — raw RNS format needed to call `isUserTenantAdmin()` |

Both are stamped at XR creation time. The label is used for list filtering. The
annotation is used for delete ownership checks and in-flight workflow tenant matching.

### 3. Tenant Identity + Admin Verification

Tenant context flows from `X-Tenant-ID` header (browser: derived from selected tenant,
CLI: explicitly passed). The API server verifies admin status for every mutating
operation:

```go
// horizon_handler.go
func (s *APIServer) isUserTenantAdmin(r *http.Request, tenantID string) (bool, error)
```

This calls the Horizon API using the caller's Keycloak access token extracted from
the session. Only verified tenant admins can create, list-filtered, or delete resources.

---

## Authentication Architecture

### BFF Pattern

The API server implements the Backend For Frontend pattern directly:

```
Browser (React :3000)
  │  Cookie: session=<random hex ID>    <- not a JWT, just a DB lookup key
  ▼
Vite dev proxy (:3000 -> :8080)         <- dev only, no logic
  ▼
Go API Server (:8080)
  │
  ├── SessionMiddleware (bff_auth.go)
  │     1. reads session cookie
  │     2. looks up session ID in PostgreSQL
  │     3. decrypts access_token + refresh_token (AES-GCM)
  │     4. auto-refreshes JWT if expired (via Keycloak)
  │     5. injects Principal{AccessToken, UserID, ...} into request context
  │
  └── handler
        uses principal.AccessToken for Horizon API calls (isUserTenantAdmin)
```

JWTs never reach the browser. The session cookie is a random opaque ID scoped to the
API server session store (PostgreSQL).

### Login Flow (PKCE)

```
Browser -> GET /auth/login
  -> redirect to Keycloak with code_challenge (S256 PKCE)
  -> Keycloak -> GET /auth/callback?code=...
  -> API server exchanges code -> gets access_token + refresh_token
  -> encrypts both (AES-GCM), stores in DB
  -> sets HttpOnly cookie: session=<random hex>
  -> redirects browser to /
```

---

## Implementation Status

### What Is Already Done (Pre-MCUCP-192)

| Feature | Status | File |
|---|---|---|
| Keycloak OIDC + BFF session | Done | `bff_auth.go` |
| Horizon API tenant admin verification | Done | `horizon_handler.go` |
| GCP credential upload + ProviderConfig auto-creation | Done | `settings_handler.go` |
| Per-tenant Ed25519 JWK + on-demand JWT for Omnia | Done | `settings_handler.go` |
| ProviderConfig injection on XR creation (compute, storage, lb) | Done | `compute_handler.go`, `storage_handler.go`, `load_balancer_handler.go` |
| Tenant admin check on all create operations (compute, storage, lb) | Done | same |

### MCUCP-192 Changes (API-Layer Isolation Gaps)

#### Gap 2 — Tenant Label Stamping on XR Creation

**Problem:** No handler stamped ownership labels on XRs. Without labels, there was
nothing to filter on in list endpoints and no way to verify ownership on delete.

**Fix:** Added label + annotation in every create handler:

```go
// In createXDatabaseYAML(), createXKubernetesClusterYAML()
annotations["platform.ucp.io/tenant-id"] = req.TenantID           // raw RNS
metadata["labels"] = map[string]interface{}{
    "platform.ucp.io/tenant": sanitizeTenantID(req.TenantID),      // sanitized
}
```

Affected: `main.go` (XDatabase), `kubernetes_handler.go` (XKubernetesCluster).
Note: compute, storage, and load balancer handlers already had this pattern.

#### Gap 1 — List Endpoints Return All Tenants' Resources

**Problem:** All list handlers fetched all cluster resources with no tenant filter.
Any authenticated user could call `GET /api/v1/databases` and receive every
tenant's resources.

**Fix:** Accept optional `?tenantId=` query param. When present, verify admin status
then filter:

```go
tenantID := strings.TrimSpace(r.URL.Query().Get("tenantId"))
if tenantID != "" {
    if ok, err := s.isUserTenantAdmin(r, tenantID); !ok { return 403 }
}
// ...
for _, item := range list.Items {
    if tenantID != "" && !xrBelongsToTenant(&item, tenantID, env) {
        continue
    }
}
```

The `xrBelongsToTenant` helper checks three fallbacks in order:

```go
// main.go
func xrBelongsToTenant(obj *unstructured.Unstructured, tenantID, env string) bool {
    // 1. annotation (raw RNS comparison)
    if obj.GetAnnotations()["platform.ucp.io/tenant-id"] == tenantID {
        return true
    }
    // 2. label (sanitized comparison)
    if obj.GetLabels()["platform.ucp.io/tenant"] == sanitizeTenantID(tenantID) {
        return true
    }
    // 3. providerConfig name (legacy resources without labels)
    providerConfig, _, _ := unstructured.NestedString(obj.Object, "spec", "parameters", "providerConfig")
    return providerConfig == gcpProviderConfigName(tenantID, env)
}
```

The third fallback covers resources provisioned before label stamping was added.

In-flight Temporal workflows are also filtered via `wf.TenantID`.

Affected: `ListDatabases()`, `ListKubernetesClusters()`.
Note: compute, storage, and load balancer list handlers already used
`computeInstanceBelongsToTenant` / `loadBalancerBelongsToTenant` — same 3-step
logic, separate functions (not refactored per YAGNI).

#### Gap 4 — Delete Has No Tenant Ownership Check

**Problem:** Delete endpoints deleted any named resource without checking whether
it belonged to the requesting tenant.

**Fix:** Read XR first, extract tenant ID from annotation, verify admin before deleting:

```go
obj, err := s.k8sClient.Resource(gvr).Get(ctx, name, metav1.GetOptions{})
if err != nil {
    if k8serrors.IsNotFound(err) { return 404 }
    return 500
}
xrTenantID := obj.GetAnnotations()["platform.ucp.io/tenant-id"]
if xrTenantID == "" {
    xrTenantID, _, _ = unstructured.NestedString(obj.Object, "spec", "parameters", "tenantId")
}
if xrTenantID != "" {
    if ok, err := s.isUserTenantAdmin(r, xrTenantID); !ok { return 403 }
}
```

Returns 403 (not 404) on ownership mismatch — the resource name is already known
to the caller since they just provided it. 404-on-mismatch is reserved for get-by-name
where the caller should not know if the resource exists.

Affected: `DeleteDatabase()`, `DeleteKubernetesCluster()`.

### Out of Scope

| Item | Reason |
|---|---|
| Terraform endpoints (`/api/v1/terraform/*`) | Generate random strings/passwords via OpenTofu `random` provider — no cloud credentials, no per-tenant ProviderConfig involved |
| Omnia-specific isolation | Omnia uses on-demand JWT per tenant — isolation already handled at auth layer |

---

## Remaining Work

### Frontend: Global Tenant Context (MCUCP-192, not yet done)

**Problem:** List endpoints currently return everything when no `?tenantId=` is passed.
The UI never sends `tenantId` on list requests. Requiring it as a query param would
mean adding a tenant selector to every list component — scattered changes.

**Fix:** Two-part change:

1. **Frontend** — create a global `TenantContext` at app root that stores the currently
   selected tenant. Update `useAuthFetch.ts` to inject `X-Tenant-ID: <tenant.rns>`
   header on every request automatically:

   ```ts
   // useAuthFetch.ts
   const { selectedTenant } = useTenantContext()
   if (selectedTenant) {
       headers.set('X-Tenant-ID', selectedTenant.rns)
   }
   ```

2. **Backend** — list handlers read `X-Tenant-ID` as fallback when query param absent:

   ```go
   tenantID := strings.TrimSpace(r.URL.Query().Get("tenantId"))
   if tenantID == "" {
       tenantID = strings.TrimSpace(r.Header.Get("X-Tenant-ID"))
   }
   ```

This means no changes needed in individual list components — the tenant propagates
automatically through `authFetch`.

### RBAC Role Model (MCUCP-191, not yet done)

Currently the only role check is `isUserTenantAdmin()` — binary: admin or 403.

The full 5-role model (`docs/architecture/RBAC.md`) needs to be implemented:

| Role | Permitted operations |
|---|---|
| `platform-admin` | All operations across all tenants |
| `tenant-admin` | All operations within their tenant |
| `deployer` | Create + delete resources within their tenant |
| `approver` | Approve/reject Temporal workflows |
| `viewer` | Read-only within their tenant |

Depends on MCUCP-192 label infrastructure being in place (now done).

### ValidatingAdmissionPolicy (MCUCP-192, Gap 6, not yet done)

Defense-in-depth at the Kubernetes API layer. Even if the API server is bypassed,
XRs without proper tenant labels or with mismatched ProviderConfig would be rejected:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
spec:
  validations:
    - expression: "has(object.metadata.labels['platform.ucp.io/tenant'])"
      message: "XR must have platform.ucp.io/tenant label"
```

Requires Kubernetes 1.30+ (GA). No extra tooling needed.

---

## Verification Guide

### Gap 2 — Label Stamping

Create a resource via UI, then inspect:

```bash
kubectl get xdatabase <name> -o jsonpath='{.metadata.labels}' | jq .
# {"platform.ucp.io/tenant": "rns-roc-iam--clsd-ucp"}

kubectl get xdatabase <name> -o jsonpath='{.metadata.annotations}' | jq .
# {"platform.ucp.io/tenant-id": "rns:roc:iam::clsd-ucp"}
```

### Gap 1 — List Filtering

```bash
COOKIE="Cookie: session=<value-from-browser-devtools>"
TENANT="rns:roc:iam::clsd-ucp"

# Real tenant -> 200, filtered results
curl -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/databases?tenantId=$TENANT"

# Fake tenant you are NOT admin of -> 403
curl -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/databases?tenantId=rns:roc:iam::fake-tenant"
```

### Gap 4 — Delete Ownership

```bash
# Stamp a fake tenant on an existing resource
kubectl annotate xdatabase <name> \
  platform.ucp.io/tenant-id=rns:roc:iam::fake-tenant --overwrite

# Delete via API -> 403
curl -X DELETE -H "$COOKIE" -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/databases/<name>"

# Restore
kubectl annotate xdatabase <name> \
  platform.ucp.io/tenant-id=$TENANT --overwrite
```

### Negative Test: List Without Tenant Filtering

Without a tenant context (before the frontend TenantContext work):

```bash
# No tenantId -> returns ALL resources (known gap, pending frontend fix)
curl -H "$COOKIE" "http://localhost:8080/api/v1/databases"
```

After the TenantContext work, this will return only the selected tenant's resources
because the browser will automatically send `X-Tenant-ID`.

---

## Implementation Plan (Full)

| # | Work Item | Ticket | Status |
|---|---|---|---|
| 1 | ProviderConfig auto-creation on GCP credential upload | — | Done |
| 2 | Per-tenant JWK + JWT generation for Omnia | — | Done |
| 3 | Tenant label + annotation stamping on XDatabase, XKubernetesCluster XRs | MCUCP-192 | Done |
| 4 | Tenant-filtered list endpoints (database, kubernetes) | MCUCP-192 | Done |
| 5 | Delete ownership check (database, kubernetes) | MCUCP-192 | Done |
| 6 | Global `TenantContext` + `X-Tenant-ID` injection in `useAuthFetch` | MCUCP-192 | Pending |
| 7 | Backend list handlers read `X-Tenant-ID` header as fallback | MCUCP-192 | Pending |
| 8 | Full RBAC role model — replace binary `isUserTenantAdmin()` | MCUCP-191 | Pending |
| 9 | `ValidatingAdmissionPolicy` for tenant label + ProviderConfig enforcement | MCUCP-192 | Pending |
| 10 | Change `provider-roc` OmniaDatabase to `scope=Namespaced` | — | Unblocked, not started |
| 11 | Resume namespace-per-tenant (MCUCP-119) | MCUCP-119 | Blocked on provider-upjet-gcp + provider-upjet-azure |

---

## Related

- `MCUCP-192` — this PoC implementation
- `MCUCP-191` — RBAC role model (depends on MCUCP-192 label infrastructure)
- `MCUCP-119` — namespace-scoped XRDs branch (on hold)
- `docs/architecture/RBAC.md` — full 5-role RBAC design
- `docs/architecture/BFF_AUTH.md` — authentication architecture
