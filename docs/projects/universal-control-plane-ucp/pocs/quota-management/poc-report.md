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

### Some resource types have no programmatic quota metric

Not all GCP resource count limits are exposed as programmatic metrics. Cloud SQL
instances per project is a confirmed example — it is a soft limit managed via GCP
support cases with no metric ID in Cloud Monitoring or the Cloud Quotas API. The
pre-provision gate for database creation therefore **fails open**.

The Cloud SQL metrics that do appear in Cloud Monitoring (`connect`, `get`, `list`,
`mutate`) are rate quotas on Admin API call frequency, not resource count quotas, and
are not relevant to provisioning decisions.

This is a GCP platform constraint. The platform soft quota layer is the mitigation —
it enforces UCP-level per-tenant resource count limits independently of what GCP
exposes, covering any resource type regardless of whether GCP provides a quota metric
for it.

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
entitlements regardless of what GCP allows — and crucially, it covers resource types
where the cloud provider exposes no programmatic quota metric (Cloud SQL instance count
being a confirmed example). The `quota_policies` table design
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

1. **UCP platform soft quota design** — the PoC proves that a quota data source exists
   for GCP and that the `QuotaProvider` interface is the right abstraction. The platform
   soft quota layer itself — schema, enforcement mechanism, who sets limits, quota
   increase workflow, display design, and equivalent data sources for other cloud
   providers — will be worked out during MVP implementation.

2. **Quota data fetching strategy** — five approaches are viable (proactive background
   sync, TTL cache, pure on-demand, lazy fetch with keep-warm, tiered Redis+DB+provider).
   No approach has been chosen. The decision affects gate reliability, infrastructure cost,
   data freshness, and whether the pre-provision gate should call the cloud provider
   directly or read from a local store. See [Quota Data Fetching Strategy](./ucp-quota-design.md)
   for full analysis.

---

## 5. Recommendations

**Decision: Go**

GCP quota visibility is functional and the pre-provision gate correctly blocks
over-provisioning for resource types with metric IDs. The `QuotaProvider` interface
provides a clean extension point for additional providers.

**Risks:**

| Risk | Severity | Note |
|---|---|---|
| Cloud Monitoring data freshness — metrics are time-series with ~24h alignment; quota data can be up to 24 hours stale | Low | Quota changes are infrequent; acceptable for provisioning decisions |
| Pre-provision gate fails open if Cloud Monitoring is unavailable | Medium | The current on-demand approach ties gate reliability to Cloud Monitoring uptime. Approaches that store quota data locally (Options A and D in the fetching strategy) mitigate this — but calling the cloud provider directly at provisioning time is arguably more accurate and avoids data freshness issues in UCP's own store. The right mitigation depends on which fetching strategy is chosen. |
| Quota fetch cost grows with providers and services | Medium | Current on-demand approach: `providers × services × 2` calls per request, all sequential. Cost grows linearly as UCP adds providers and resource types. See [Quota Data Fetching Strategy](./ucp-quota-design.md) for long-term approach. |

**Next steps:**

1. Design and implement the UCP platform soft quota layer during MVP implementation
   (Open Question 1)
2. Decide on the quota data fetching strategy before MVP implementation — the choice
   affects the pre-provision gate design and infrastructure requirements (Open Question 2)
3. Identify equivalent quota data sources for other cloud providers as they are
   onboarded — the `QuotaProvider` interface is ready for new implementations

---

## 6. References

- Design docs: [UCP Quota Design](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6681010042/UCP+Quota+Design)
- GCP API reference: [GCP API Reference](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6681206350/GCP+Quota+API+Reference)
- PRs: [ucp-platform #68](https://ghe.rakuten-it.com/clsd-ucp/ucp-platform/pull/68) · [ucp-api-gateway #22](https://ghe.rakuten-it.com/clsd-ucp/ucp-api-gateway/pull/22) · [ucp-ui #17](https://ghe.rakuten-it.com/clsd-ucp/ucp-ui/pull/17)
- Jira: [MCUCP-189](https://jira.rakuten-it.com/jira/browse/MCUCP-189)
- Prerequisite: [MCUCP-192 — Tenant Isolation](https://jira.rakuten-it.com/jira/browse/MCUCP-192)
