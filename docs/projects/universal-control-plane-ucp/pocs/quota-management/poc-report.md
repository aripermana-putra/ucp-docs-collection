---
title: "POC Report — Quota Management"
space: UCP
parent_page_id: "../quota-management.md"
---

# Quota Management

| | |
|---|---|
| **Jira** | [MCUCP-189](https://jira.rakuten-it.com/jira/browse/MCUCP-189) |
| **Author** | aripermana.putra |
| **Date** | 2026-06-02 |
| **Status** | COMPLETED |

---

## 1. Summary

MCUCP-189 proves that UCP can surface real-time GCP quota data per tenant and gate
resource provisioning against it. Quota limits and usage are both read from Cloud
Monitoring — no separate quota API is required. A `QuotaProvider` interface abstracts
the GCP implementation behind a provider-agnostic contract, making other cloud providers
addable without changing the API layer.

UCP quota management has two layers: **GCP cloud quota** (visibility + pre-provision
gate) and **UCP platform soft quota** (UCP-enforced per-tenant resource count limits).
The GCP cloud quota layer is functionally complete for this PoC. The platform soft quota
layer is designed but not implemented — it is deferred as a follow-on.

**Verdict: Go.** GCP quota visibility and pre-provision gating work end-to-end. The
platform soft quota design is sound and ready to implement once GCP cloud quota is
confirmed sufficient for MVP needs.

---

## 2. Objectives & Success Criteria

**Hypothesis:**
UCP can surface real-time GCP quota data per tenant using existing stored credentials,
and block resource creation before a quota-exceeded request reaches the cloud provider.

**Success criteria:**

| # | Criterion | Result |
|---|---|---|
| SC-1 | `GET /api/v1/quota` returns real-time quota limits and usage from GCP | Pass |
| SC-2 | Quota data covers all four UCP resource types: Cloud SQL, Compute Engine, GKE, Cloud Storage | Pass |
| SC-3 | Pre-provision gate returns HTTP 429 before workflow creation when quota is exhausted | Pass |
| SC-4 | `QuotaProvider` interface abstracts GCP-specific logic — new providers implement the interface | Pass |
| SC-5 | Quota data uses the tenant's stored credentials — no cross-tenant access | Pass |

**Scope boundaries (out of scope):**
- UCP platform soft quota implementation (designed, not implemented in this PoC)
- Quota increase requests via GCP Cloud Quotas API
- Pre-provision gates for resource types other than Cloud SQL
- Other cloud providers (AWS, Azure, Omnia)
- Quota alerting or threshold notifications

---

## 3. Findings

### Cloud Monitoring as the single data source

Both quota limits and usage are read from Cloud Monitoring
(`serviceruntime.googleapis.com`). No separate quota API is used. Cloud Monitoring
exposes two metric types:
- `quota/limit` — the configured limit per quota metric per region
- `quota/allocation/usage` — current consumption

These are joined on `(quota_metric, location)` to produce a usage percentage per row.
The same tenant credentials used for provisioning are sufficient to call Cloud Monitoring
— no additional credential setup is needed.

### Cloud SQL instance count has no metric ID

Cloud SQL instances per project is a **soft limit** managed via GCP support cases. It
has no metric ID in any quota API and does not appear in Cloud Monitoring. The
pre-provision gate for database creation therefore **fails open** — it cannot check
Cloud SQL instance quota before provisioning.

The Cloud SQL metrics that do appear in Cloud Monitoring (`connect`, `get`, `list`,
`mutate`) are rate quotas on Admin API call frequency, not resource count quotas. They
are not relevant to provisioning decisions.

This is a GCP platform constraint, not a UCP implementation gap. The UCP platform soft
quota layer (see below) is the mitigation — it enforces a UCP-level database count limit
independently of GCP quota visibility.

### Cloud Monitoring returns fewer metrics than the GCP Console

The GCP Console Quotas page shows more quota metrics per service than Cloud Monitoring
returns. For Cloud SQL in particular, Cloud Monitoring returns only the Admin API rate
quotas. This gap exists at the GCP platform level and affects what UCP can surface
without workarounds.

### Two-layer quota model

| Layer | What it controls | Status |
|---|---|---|
| GCP cloud quota | Real-time quota limits and usage from the tenant's GCP project | Implemented |
| UCP platform soft quota | UCP-enforced per-tenant resource count limits (`quota_policies` table + middleware) | Designed, not implemented |

The platform soft quota layer is independent of GCP quotas. It enforces UCP-level
entitlements regardless of what GCP allows — and crucially, it can gate Cloud SQL
creation even though GCP provides no metric for it. The `quota_policies` table design
and the `CheckQuota` middleware contract are specified in the design doc.

### Project-per-tenant means independent GCP quotas

With UCP's ProviderConfig-per-tenant model, each tenant's GCP credentials point to a
dedicated cloud project. GCP quotas are per-project with no cross-project pooling. This
means each tenant's GCP quota is fully independent — one tenant's resource consumption
does not affect another's quota.

The platform soft quota should be set below the GCP project quota to leave headroom and
prevent GCP-level failures.

---

## 4. Open Questions

1. **Who sets initial platform quotas?** Platform-admin via API, or seeded from Horizon
   tenant tier (`standard`/`premium`)? The tier field suggests Horizon already encodes
   capacity intent.

2. **Cross-provider quota** — should UCP enforce a single limit across all providers
   (e.g. 5 databases total regardless of GCP or Omnia), or per-provider limits
   (e.g. 5 GCP databases + 5 Omnia databases)?

3. **Quota increase workflow** — should tenants be able to request a platform quota
   increase that triggers an approval workflow, optionally followed by a GCP
   `QuotaPreference` submission?

4. **Platform quota drift when resources are deleted outside UCP** — if a resource is
   deleted directly in GCP, the platform ledger count is wrong until the next
   reconciliation. This connects to the drift detection work (MCUCP-158).

5. **Rate quotas vs resource count quotas in the display** — they serve different
   audiences (operations vs capacity planning). Should they be shown together or
   separated?

---

## 5. Recommendations

**Decision: Go**

GCP quota visibility is functional and the pre-provision gate correctly blocks
over-provisioning for resource types with metric IDs. The `QuotaProvider` interface
provides a clean extension point for additional providers.

**Next steps:**

1. Implement the UCP platform soft quota layer (`quota_policies` + `CheckQuota`
   middleware) — this is the correct mitigation for the Cloud SQL instance count gap
   and the foundation for cross-provider quota enforcement
2. Align with PM on initial quota values, cross-provider model, and quota increase
   workflow (Open Questions 1, 2, 3)
3. Wire pre-provision gates for Compute Engine, GKE, and Cloud Storage once the
   quota metric mappings are confirmed

---

## 6. References

- Design docs: [UCP Quota Design](./ucp-quota-design.md)
- GCP API reference: [GCP API Reference](./gcp-api-reference.md)
- PRs: [ucp-platform #68](https://ghe.rakuten-it.com/clsd-ucp/ucp-platform/pull/68) · [ucp-api-gateway #22](https://ghe.rakuten-it.com/clsd-ucp/ucp-api-gateway/pull/22) · [ucp-ui #17](https://ghe.rakuten-it.com/clsd-ucp/ucp-ui/pull/17)
- Jira: [MCUCP-189](https://jira.rakuten-it.com/jira/browse/MCUCP-189)
- Prerequisite: [MCUCP-192 — Tenant Isolation](https://jira.rakuten-it.com/jira/browse/MCUCP-192)
