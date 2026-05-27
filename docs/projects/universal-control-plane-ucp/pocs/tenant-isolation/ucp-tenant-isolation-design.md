---
title: "Tenant Isolation — UCP Design"
space: UCP
parent_page_id: "../tenant-isolation.md"
---

# Tenant Isolation — UCP Design

This document covers the design decisions behind global tenant context propagation,
the RBAC role model, Kubernetes admission enforcement, and the long-term path to
namespace-per-tenant isolation.

---

## Global Tenant Context and Tenant Scope Propagation

### Design

A global `TenantContext` is maintained at the React app root. It holds the currently
selected tenant and exposes it via a `useTenantContext()` hook. Tenant scope is
propagated differently per HTTP method, following REST conventions:

**GET list — `?tenantId=` query param**

`useAuthFetch` appends `?tenantId=<tenant.rns>` to GET requests when a tenant is
selected. Tenant scope is part of the query, not a custom header, making it
visible in URLs, logs, and compatible with any HTTP client without custom header support.

```ts
// useAuthFetch.ts
const { selectedTenant } = useTenantContext()
const isGet = !options.method || options.method.toUpperCase() === 'GET'
if (selectedTenant && isGet) {
    const sep = url.includes('?') ? '&' : '?'
    finalUrl = `${url}${sep}tenantId=${encodeURIComponent(selectedTenant.rns)}`
}
```

List handlers on the API server read from the query param:

```go
tenantID := strings.TrimSpace(r.URL.Query().Get("tenantId"))
```

When `?tenantId=` is absent, the handler calls Horizon to fetch all tenants the caller
belongs to and returns resources across all of them.

**POST create — `tenantId` in request body**

Tenant context is included as a field in the JSON body. The frontend create forms
read `selectedTenant.rns` and include it as `tenantId`. No per-request URL manipulation
is needed.

**DELETE / approve / reject — `{tenantSlug}` path segment**

Mutation endpoints include the tenant slug as the first path segment:

```
DELETE /api/v1/tenants/{tenantSlug}/databases/{name}
POST   /api/v1/tenants/{tenantSlug}/workflows/{id}/approve
```

The slug (e.g. `clsd-ucp`) is the URL-safe short name stored in the `tenants.slug`
column. The API resolves it to the canonical RNS via a DB lookup before the ownership
check. Frontend list components construct the URL using `selectedTenant.name` (the slug).

```ts
// list component delete call
authFetch(`/api/v1/tenants/${selectedTenant.name}/storage/${resource.name}`,
          { method: 'DELETE' })
```

---

## RBAC Role Model (MCUCP-191)

The current implementation uses a single check — `isUserTenantAdmin()` — for all
protected operations. The full 5-role per-tenant model replaces this binary check with
per-operation permission enforcement.

### Roles

| Role | Scope | Description |
|---|---|---|
| `platform-admin` | Platform | UCP operators. Full access across all tenants. |
| `tenant-admin` | Tenant | Full access within their tenant. |
| `deployer` | Tenant | Create and delete resources within their tenant. |
| `approver` | Tenant | Approve and reject Temporal workflows. |
| `viewer` | Tenant | Read-only access within their tenant. |

### Permission Mapping

| Operation | Minimum required role |
|---|---|
| Create resource | `deployer` |
| List resources (own tenant) | `viewer` |
| Get resource by name (own tenant) | `viewer` |
| Delete resource | `deployer` |
| Approve / reject workflow | `approver` |
| Upload credentials | `tenant-admin` |
| List resources (any tenant) | `platform-admin` |

### Implementation Approach

A `RequireRole(minRole string)` middleware resolves the caller's role from the UCP
`tenant_role_assignments` database table and rejects requests that do not meet the
minimum required role for the endpoint.

For GET requests, role resolution uses the `?tenantId=` query param. For mutation
endpoints with `{tenantSlug}` in the path, the slug is resolved to the canonical RNS
before the role check. See the RBAC PoC documentation for the full role model design.

---

## ValidatingAdmissionPolicy

A `ValidatingAdmissionPolicy` at the Kubernetes API layer enforces that:
- All XRs carry the `platform.ucp.io/tenant` label
- The `spec.parameters.providerConfig` value matches the naming convention for the
  declared tenant label

This provides defense-in-depth: even if the API server is bypassed, a rogue XR cannot
be created without proper ownership metadata.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-tenant-label
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["platform.example.io"]
        apiVersions: ["v1alpha1"]
        resources: ["*"]
        operations: ["CREATE", "UPDATE"]
  validations:
    - expression: "has(object.metadata.labels) && 'platform.ucp.io/tenant' in object.metadata.labels"
      message: "XR must carry the platform.ucp.io/tenant label"
    - expression: "has(object.metadata.annotations) && 'platform.ucp.io/tenant-id' in object.metadata.annotations"
      message: "XR must carry the platform.ucp.io/tenant-id annotation"
```

`ValidatingAdmissionPolicy` is GA from Kubernetes 1.30. No additional tooling is required.

---

## Namespace Per Tenant (Long-Term Path)

Namespace-per-tenant is the correct long-term isolation model for two reasons.

**RBAC isolation at the Kubernetes layer.** Kubernetes RBAC can restrict access to
namespaced resources by namespace, providing isolation enforced by the Kubernetes API
server independent of the UCP API server. With cluster-scoped XRs this is not possible —
cluster-scoped resources have no namespace boundary for RBAC to act on.

**List performance.** All current list endpoints fetch every XR of a given type
cluster-wide and filter tenant ownership in the UCP API server's memory:

```go
// Today: fetches all tenants' resources, filters in UCP API server memory
list, err := s.k8sClient.Resource(xObjectStorageGVR).List(ctx, metav1.ListOptions{})
for _, item := range list.Items {
    if !xrBelongsToAnyTenant(&item, allowedTenantIDs, env) {
        continue
    }
}
```

With namespace-scoped XRs, the list call is scoped to the tenant's namespace and
Kubernetes filters server-side using its namespace index:

```go
// After MCUCP-119: only this tenant's resources returned, no in-memory filtering needed
list, err := s.k8sClient.Resource(xObjectStorageGVR).Namespace(tenantNamespace).List(ctx, metav1.ListOptions{})
```

This eliminates the `getUserTenantIDs` Horizon call, the `xrBelongsToAnyTenant`
in-memory loop, and the three-fallback ownership check on every list request. Memory
pressure and latency in the UCP API server grow with the requesting tenant's resource
count rather than with the total across all tenants.

The approach is blocked until all required providers ship namespace-scoped managed resource
(MR) support:

| Provider | MR Scope | Namespace Support |
|---|---|---|
| provider-upjet-gcp | Cluster | In progress (upstream) |
| provider-upjet-aws | Namespace | Available in v2 |
| provider-upjet-azure | Cluster | In progress (upstream) |
| provider-roc (Omnia) | Cluster | Unblocked — owned by platform team |
| provider-terraform | Cluster | Not started |

`provider-roc` can be migrated to namespace-scoped independently of the upjet providers.
The migration requires changing `scope: Cluster` to `scope: Namespaced` in the
`OmniaDatabase` CRD and updating the reconciler to scope operations to the MR's namespace.

Once all required providers support namespaced MRs, MCUCP-119 resumes: XRDs change from
`scope: Cluster` to `scope: Namespaced`, and each tenant gets a dedicated Kubernetes
namespace that Crossplane resources are provisioned into.
