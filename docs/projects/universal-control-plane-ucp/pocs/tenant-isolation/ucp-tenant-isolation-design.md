---
title: "Tenant Isolation — UCP Design"
space: UCP
parent_page_id: "../tenant-isolation.md"
---

# Tenant Isolation — UCP Design

This document covers the design for isolation features not yet deployed: global tenant
context propagation, the RBAC role model, Kubernetes admission enforcement, and the
long-term path to namespace-per-tenant isolation.

---

## Global Tenant Context and Automatic Header Injection

### Problem

List endpoints enforce tenant filtering only when `?tenantId=` is explicitly passed.
The browser UI sends no tenant ID on list requests, so all resources across all tenants
are returned to any authenticated user.

### Design

A global `TenantContext` is maintained at the React app root. It holds the currently
selected tenant and exposes it via a `useTenantContext()` hook. The `useAuthFetch` hook
reads from this context and injects `X-Tenant-ID: <tenant.rns>` on every outgoing
request — no changes are needed in individual list components.

```ts
// useAuthFetch.ts
const { selectedTenant } = useTenantContext()
if (selectedTenant) {
    headers.set('X-Tenant-ID', selectedTenant.rns)
}
```

List handlers on the API server read `X-Tenant-ID` as a fallback when the query
parameter is absent:

```go
tenantID := strings.TrimSpace(r.URL.Query().Get("tenantId"))
if tenantID == "" {
    tenantID = strings.TrimSpace(r.Header.Get("X-Tenant-ID"))
}
```

The query parameter takes precedence over the header, preserving the ability to override
for admin use cases.

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

A `RequireRole(minRole string)` middleware reads the caller's roles from the Horizon API
(or a cached token claim) and rejects requests that do not meet the minimum required role
for the endpoint. All endpoints currently guarded by `isUserTenantAdmin()` are migrated
to `RequireRole("tenant-admin")` as a starting point.

Role resolution requires the tenant context to be available on the request — provided by
`X-Tenant-ID` or `?tenantId=`. This is why the global tenant context work is a
prerequisite.

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
    - expression: "has(object.metadata.labels['platform.ucp.io/tenant'])"
      message: "XR must carry the platform.ucp.io/tenant label"
    - expression: "has(object.metadata.annotations['platform.ucp.io/tenant-id'])"
      message: "XR must carry the platform.ucp.io/tenant-id annotation"
```

`ValidatingAdmissionPolicy` is GA from Kubernetes 1.30. No additional tooling is required.

---

## Namespace Per Tenant (Long-Term Path)

Namespace-per-tenant is the correct long-term isolation model. It allows Kubernetes RBAC
to enforce isolation at the API-server level independent of the UCP API server.

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
