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
| **Date** | 2026-05-29 |
| **Status** | COMPLETED |

---

## 1. Summary

MCUCP-192 proves that UCP can enforce per-tenant resource isolation across a shared
Kubernetes cluster without namespace-per-tenant. Three mechanisms work in concert:
ownership label stamping at XR creation, Kubernetes server-side label filtering on
list endpoints, and ownership verification on all mutations. A `ValidatingAdmissionPolicy`
provides defense-in-depth at the cluster level independent of the API server.

**Verdict: Go.** Tenant isolation is functionally complete for all current resource types.
Namespace-per-tenant is the correct long-term path but requires both upstream provider
support and a ProviderConfig admission policy layer before it can be safely adopted.

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
- Namespace-per-tenant — blocked on upstream provider MR support and ProviderConfig hardening design
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
a rogue XR created by bypassing the API server (e.g. via direct `kubectl apply` from
a compromised platform team account) is rejected at the Kubernetes admission layer
before persisting to etcd.

### ProviderConfig injection

The `ProviderConfig` name is computed server-side from the tenant ID and environment:
`gcp-{sanitized-tenant-id}-{env}`. Callers cannot supply or override it. This ensures
every XR for a tenant uses that tenant's cloud credentials — cross-tenant provisioning
via a crafted `providerConfig` value is not possible.

### Namespace-per-tenant requires more than namespaced provider MRs

Cluster-scoped XRs have no namespace boundary for Kubernetes RBAC to act on.
Namespace-per-tenant would provide native Kubernetes isolation and eliminate the
`getUserTenantIDs` Horizon call on list endpoints. The commonly understood blocker
is that provider MRs must be namespace-scoped first (provider-upjet-gcp and
provider-upjet-azure are still cluster-scoped).

However, research into the Crossplane threat model reveals an additional design
requirement. Namespaced ProviderConfigs provide namespace isolation for the
*reference graph* (MR → PC → Secret) but do **not** restrict the provider's
credential configuration surface. A ProviderConfig can specify:

- `credentials.source: InjectedIdentity` — the controller uses its own ambient cloud
  identity (the operator's service account) instead of a tenant-supplied Secret
- `endpoint: <attacker-controlled-url>` — the controller sends authenticated requests
  to an attacker's server

Combined, these allow a tenant (or a compromised operator) to craft a PC that causes
the Crossplane controller to send its own cloud credentials to an attacker-controlled
endpoint — an SSRF-via-configuration attack. The Crossplane runtime cannot prevent
this because these fields are provider-specific and semantically opaque to the runtime.

The Crossplane maintainers confirm this is the intended contract: namespaced PCs
are safe only when operators constrain the credential configuration surface via
admission policy. A `ValidatingAdmissionPolicy` or equivalent must block
`InjectedIdentity` sources and custom endpoint overrides on all tenant-namespace
ProviderConfigs before namespace-per-tenant can be safely adopted.

This applies even when UCP controls PC creation (no direct tenant access) — it
provides defense-in-depth against a compromised platform team account.

---

## 4. Open Questions

1. **Namespace-per-tenant** — the correct long-term isolation model. Two blockers
   must be resolved before it can be safely adopted:
   - All required provider MRs must become namespace-scoped (provider-upjet-gcp is still cluster-scoped)
   - A `ValidatingAdmissionPolicy` for ProviderConfigs must block `InjectedIdentity`
     credential sources and custom endpoint overrides in tenant namespaces

---

## 5. Recommendations

**Decision: Go**

Label-based tenant isolation is functionally complete for all current resource types.
The `ValidatingAdmissionPolicy` provides a meaningful safety net at the cluster layer.
The ProviderConfig injection model correctly enforces per-tenant cloud credential scope.

The current approach remains in place until provider-upjet-gcp ships namespace-scoped
MR support. No action is required to unblock that — it is an upstream dependency.

**Risks of the current implementation:**

| Risk | Severity | Mitigation |
|---|---|---|
| All isolation relies on the UCP API server — no independent Kubernetes-layer enforcement. A bug in permission middleware or auth bypass breaks all tenant isolation. | High | Route-level permission enforcement; parameterized queries; tenant-scoped DB queries; audit logging. Layered with Kubernetes RBAC least-privilege for cluster access — direct cluster access should be restricted and require explicit justification, not be routine. |
| The `ValidatingAdmissionPolicy` enforces label *presence* but not label *ownership* — a platform team member with direct cluster access could stamp any tenant's label on a manually created XR. | Medium | Restrict direct cluster access via Kubernetes RBAC least-privilege. Direct cluster access should be elevated/break-glass, not a day-to-day capability, to limit blast radius of a compromised account. |
| **[Future — namespace-per-tenant]** ProviderConfig SSRF — once tenants or operators have namespace write access, a crafted PC with `InjectedIdentity` + custom `endpoint` could cause the Crossplane controller to exfiltrate its own ambient cloud credentials. A `ValidatingAdmissionPolicy` constraining identity-influencing fields on ProviderConfigs is required before namespace-per-tenant can be safely adopted. | Low (future) | Design and deploy ProviderConfig admission policy before namespace-per-tenant migration (Open Question 1) |

---

## 6. References

- Design docs: [Tenant Isolation — UCP Design](./ucp-tenant-isolation-design.md)
- Crossplane threat model discussion: [crossplane/crossplane#7392](https://github.com/crossplane/crossplane/discussions/7392)
- PRs: [ucp-platform #44](https://ghe.rakuten-it.com/clsd-ucp/ucp-platform/pull/44) · [ucp-api-gateway #15](https://ghe.rakuten-it.com/clsd-ucp/ucp-api-gateway/pull/15) · [ucp-ui #12](https://ghe.rakuten-it.com/clsd-ucp/ucp-ui/pull/12)
- Jira: [MCUCP-192](https://jira.rakuten-it.com/jira/browse/MCUCP-192)
- Prerequisite: [MCUCP-189 — Quota Management](https://jira.rakuten-it.com/jira/browse/MCUCP-189)
