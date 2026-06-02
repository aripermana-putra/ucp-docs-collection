---
title: "Tenant Isolation"
space: UCP
parent_page_id: "../pocs.md"
---

# Tenant Isolation

UCP enforces tenant isolation at the API server layer using a
**ProviderConfig-per-tenant** strategy combined with ownership labels on all
Kubernetes XRs. Tenants have no direct access to the Kubernetes API — the API server is
the sole isolation boundary. A `ValidatingAdmissionPolicy` at the cluster layer provides
defense-in-depth independent of the API server.

---

## Approach Summary

Each tenant's cloud credentials are stored as a Kubernetes Secret. A dedicated
`ProviderConfig` pointing to that tenant's cloud account is automatically created when
credentials are uploaded. When the API server provisions any resource for a tenant, it
injects the correct `ProviderConfig` server-side. Tenants cannot reference another
tenant's `ProviderConfig`.

All XRs created by the API server carry a `platform.ucp.io/tenant` label and a
`platform.ucp.io/tenant-id` annotation. List and delete endpoints use these markers to
enforce per-tenant scoping at the API layer. The `ValidatingAdmissionPolicy` rejects any
XR that does not carry these markers, even if created by bypassing the API server.

Namespace-per-tenant is the planned long-term enhancement — adding native Kubernetes
enforcement on top of the existing label-based isolation. It is blocked on namespace-scoped
MR support in provider-upjet-gcp.

---

## Sub-Documents

- [POC Report](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6670032897/POC+Report+Tenant+Isolation) — verdict, success criteria, findings, risks, open questions
- [Concepts](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6646697166/Tenant+Isolation+%E2%80%94+Concepts) — ProviderConfig-per-tenant, tenant identity,
  label vs annotation, BFF auth, namespace isolation constraint
- [Implementation](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6646697174/Tenant+Isolation+%E2%80%94+Implementation) — scope, API sequence diagrams, key
  functions, verification
- [Design](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6646697182/Tenant+Isolation+%E2%80%94+UCP+Design) — global tenant context,
  ValidatingAdmissionPolicy, namespace-per-tenant path

---

## Related

- `MCUCP-192` — API-layer isolation implementation
- `MCUCP-191` — RBAC role model (depends on MCUCP-192)
