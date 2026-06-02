---
title: "POC Report — Tenant Isolation"
space: UCP
parent_page_id: "../tenant-isolation.md"
---

# Tenant Isolation

| | |
|---|---|
| **Jira** | [MCUCP-192](https://jira.rakuten-it.com/jira/browse/MCUCP-192) |
| **Author** | aripermana.putra |
| **Date** | 2026-06-02 |
| **Status** | COMPLETED |

---

## 1. Summary

MCUCP-192 proves that UCP can enforce per-tenant resource isolation across a shared
Kubernetes cluster without namespace-per-tenant. Three mechanisms work in concert:
ownership label stamping at XR creation, Kubernetes server-side label filtering on
list endpoints, and ownership verification on all mutations. A `ValidatingAdmissionPolicy`
provides defense-in-depth at the cluster level independent of the API server.

**Verdict: Go.** Tenant isolation is functionally complete for all current resource types.
Namespace-per-tenant is the correct long-term path but is blocked on upstream provider
support and deferred to MCUCP-119.

---

## 2. Objectives & Success Criteria

**Hypothesis:**
UCP can enforce per-tenant resource isolation on a shared cluster using Kubernetes
label selectors and server-side ownership checks, without relying on namespace boundaries.

**Success criteria:**

| # | Criterion | Result |
|---|---|---|
| SC-1 | Resources created for tenant A are not visible in tenant B's list | Pass |
| SC-2 | Tenant B cannot delete a resource owned by tenant A | Pass |
| SC-3 | Approve/reject workflow only succeeds for the owning tenant | Pass |
| SC-4 | `providerConfig` is always server-derived — callers cannot override it | Pass |
| SC-5 | An XR created without the tenant label is rejected at the cluster level | Pass |
| SC-6 | List without `?tenantId=` returns resources across all the caller's tenants | Pass |
| SC-7 | Wrong-tenant delete returns 404 not 403 — resource existence is not leaked | Pass |

**Scope boundaries (out of scope):**
- Namespace-per-tenant (deferred to MCUCP-119 — blocked on provider MR support)
- Omnia-specific isolation — handled at the auth layer via per-tenant JWT
- Terraform endpoints — `random` provider has no cloud credentials; per-tenant ProviderConfig is not applicable

---

## 3. Findings

### Label-based ownership

Every XR carries two tenant ownership markers stamped at creation time:

- **Label** `platform.ucp.io/tenant` — sanitized RNS value (`:` replaced with `-`), used as a Kubernetes label selector
- **Annotation** `platform.ucp.io/tenant-id` — raw RNS string, used in API responses and ownership comparisons

The sanitization is required because Kubernetes label values follow DNS label rules
(RFC 1123) that do not permit `:`. The raw RNS is preserved in the annotation, which
has no character restrictions.

All list endpoints push a `platform.ucp.io/tenant in (t1, t2, ...)` label selector to
the Kubernetes API server. Filtering happens server-side using the label index — no
in-memory filtering in the API server.

### Mutation ownership enforcement

Delete, approve, and reject operations use the tenant slug as a URL path segment
(`/api/v1/tenants/{tenantSlug}/...`). The API resolves the slug to the canonical RNS
via a DB lookup, then issues a single Kubernetes List with both a field selector
(`metadata.name=<name>`) and the tenant label selector. An empty result means either
the resource does not exist or it belongs to a different tenant — both return 404,
not 403, to avoid leaking resource existence across tenants.

### ValidatingAdmissionPolicy

A Kubernetes `ValidatingAdmissionPolicy` (GA in K8s 1.30) enforces that all XRs carry
the tenant label and annotation on CREATE and UPDATE. This provides defense-in-depth:
a rogue XR created by bypassing the API server (e.g. via direct `kubectl apply`) is
rejected at the Kubernetes admission layer before persisting to etcd.

### ProviderConfig injection

The `ProviderConfig` name is computed server-side from the tenant ID and environment:
`gcp-{sanitized-tenant-id}-{env}`. Callers cannot supply or override it. This ensures
every XR for a tenant uses that tenant's cloud credentials — cross-tenant provisioning
via a crafted `providerConfig` value is not possible.

### Namespace-per-tenant is blocked

Cluster-scoped XRs have no namespace boundary for Kubernetes RBAC to act on.
Namespace-per-tenant would provide native Kubernetes isolation and eliminate the
`getUserTenantIDs` Horizon call on list endpoints, but it requires all provider MRs
to be namespace-scoped. The upjet GCP and Azure providers are still cluster-scoped
(namespace support in progress upstream). This is tracked in MCUCP-119.

---

## 4. Open Questions

1. **Namespace-per-tenant (MCUCP-119)** — the correct long-term isolation model.
   Namespace-scoped XRs would shift list filtering from label selectors to namespace
   scoping (faster, uses the namespace index), and would provide Kubernetes RBAC
   enforcement independent of the API server. Blocked until provider-upjet-gcp and
   provider-upjet-azure ship namespace-scoped MR support.

2. **Legacy XRs without ownership labels** — XRs created before label stamping are
   invisible to list and delete endpoints. A migration strategy is needed before
   namespace-per-tenant can be adopted: all existing cluster-scoped XRs must be
   labelled or the migration will silently drop them from the API.

3. **Approve/reject ownership check covers databases only** —
   `verifyWorkflowTenantOwnership` looks up the workflow ID in XDatabase annotations.
   When approval workflows are added for other resource types (compute, storage, etc.),
   the ownership check must be extended to cover those XR types.

4. **`getUserTenantIDs` Horizon call on unfiltered list** — when `?tenantId=` is absent,
   the API calls Horizon to fetch all tenants the caller belongs to. This adds a live
   Horizon call to every unfiltered list request. Namespace-per-tenant eliminates this
   entirely. Until then, the call is acceptable for PoC but worth caching or replacing
   with the locally-synced `oc_roles` table before MVP.

---

## 5. Recommendations

**Decision: Go**

Label-based tenant isolation is functionally complete for all current resource types.
The `ValidatingAdmissionPolicy` provides a meaningful safety net at the cluster layer.
The ProviderConfig injection model correctly enforces per-tenant cloud credential scope.

**Next steps:**

1. Label all existing unlabelled XRs before namespace-per-tenant migration
   (Open Question 2)
2. Extend `verifyWorkflowTenantOwnership` to cover all resource types when their
   approval workflows are added (Open Question 3)
3. Replace the `getUserTenantIDs` Horizon call on unfiltered list with the locally-synced
   `oc_roles` table (Open Question 4) — reduces live Horizon dependency before MVP
4. Resume MCUCP-119 (namespace-per-tenant) once provider-upjet-gcp ships
   namespace-scoped MR support (Open Question 1)

---

## 6. References

- Design docs: [Tenant Isolation — UCP Design](./ucp-tenant-isolation-design.md)
- PRs: [ucp-platform #44](https://ghe.rakuten-it.com/clsd-ucp/ucp-platform/pull/44) · [ucp-api-gateway #15](https://ghe.rakuten-it.com/clsd-ucp/ucp-api-gateway/pull/15) · [ucp-ui #12](https://ghe.rakuten-it.com/clsd-ucp/ucp-ui/pull/12)
- Jira: [MCUCP-192](https://jira.rakuten-it.com/jira/browse/MCUCP-192)
- Follow-up: [MCUCP-119 — Namespace-scoped Crossplane resources](https://jira.rakuten-it.com/jira/browse/MCUCP-119)
- Prerequisite: [MCUCP-189 — Quota Management](https://jira.rakuten-it.com/jira/browse/MCUCP-189)
