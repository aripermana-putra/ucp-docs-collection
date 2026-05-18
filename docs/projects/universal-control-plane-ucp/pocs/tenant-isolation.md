---
title: "Tenant Isolation"
space: UCP
parent_page_id: "../pocs.md"
---

# Tenant Isolation

UCP enforces tenant isolation at the API server layer using a
**ProviderConfig-per-tenant** strategy combined with ownership labels on all
Kubernetes XRs. Tenants have no direct access to the Kubernetes API — the API server is
the sole isolation boundary.

---

## Approach Summary

Each tenant's cloud credentials are stored as a Kubernetes Secret. A dedicated
`ProviderConfig` pointing to that tenant's cloud account is automatically created when
credentials are uploaded. When the API server provisions any resource for a tenant, it
injects the correct `ProviderConfig` server-side. Tenants cannot reference another
tenant's `ProviderConfig`.

All XRs created by the API server carry a `platform.ucp.io/tenant` label and a
`platform.ucp.io/tenant-id` annotation. List and delete endpoints use these markers to
enforce per-tenant scoping at the API layer.

---

## Sub-Documents

- [Concepts](tenant-isolation/concepts.md) — ProviderConfig-per-tenant, tenant identity,
  label vs annotation, BFF auth, namespace isolation constraint
- [Implementation](tenant-isolation/implementation.md) — scope, API sequence, key
  functions, verification
- [Design](tenant-isolation/ucp-tenant-isolation-design.md) — global tenant context,
  RBAC role model, ValidatingAdmissionPolicy, namespace-per-tenant path

---

## Related

- `MCUCP-192` — API-layer isolation implementation
- `MCUCP-191` — RBAC role model (depends on MCUCP-192)
- `MCUCP-119` — namespace-scoped XRDs (blocked on upstream provider support)
- `docs/architecture/RBAC.md` — 5-role RBAC design specification
