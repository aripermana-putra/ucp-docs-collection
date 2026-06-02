---
title: "UCP Quota Design"
space: UCP
parent_page_id: "../quota-management.md"
---

# UCP Quota Design

## Two-Layer Model

UCP quota management has two layers:

| Layer | What it controls | PoC status |
|---|---|---|
| GCP cloud quota | Real-time quota limits and usage from the tenant's GCP project | Implemented |
| UCP platform soft quota | UCP-enforced per-tenant resource count limits | Not implemented — future work |

The two layers are independent. GCP cloud quota is a hard ceiling set by Google per
project. Platform soft quota is a UCP-enforced entitlement that sits below the GCP
ceiling. If the GCP project quota is exhausted, resource creation fails at the GCP API
layer regardless of platform quota. If the platform soft quota is exhausted, UCP blocks
the request before it reaches GCP.

For UCP's ProviderConfig-per-tenant model, each tenant's GCP quota is independent —
one tenant exhausting their quota does not affect another. Platform soft quota should
be set below the GCP project quota to leave headroom.

---

## GCP Cloud Quota Layer

### Data source

Both quota limits and usage are read from Cloud Monitoring
(`serviceruntime.googleapis.com`). No separate quota API is used.

| Metric type | What it provides |
|---|---|
| `serviceruntime.googleapis.com/quota/limit` | Configured limit per quota metric per region |
| `serviceruntime.googleapis.com/quota/allocation/usage` | Current consumption |

The two are joined on `(quota_metric, location)` to produce usage percentage per row.
The tenant's stored provisioning credentials are sufficient to call Cloud Monitoring —
no additional credential setup is needed.

### Pre-provision gate

`CheckPreProvision` is called on every resource creation request before the Temporal
workflow is started. If the tenant's quota for the resource type is exhausted, it
returns HTTP 429 immediately.

| UCP resource type | GCP quota metric | Gate status |
|---|---|---|
| `database` | — (soft limit, no metric ID) | Fails open |
| `compute` | `compute.googleapis.com/cpus` | Not wired |
| `k8s-cluster` | `container.googleapis.com/clusters` | Not wired |
| `storage` | `storage.googleapis.com/buckets` | Not wired |

Not all GCP resource count limits are exposed as programmatic metrics. Cloud SQL
instance count is a confirmed example — it is a soft limit with no metric ID, causing
the database gate to fail open. The platform soft quota layer (see below) is the correct
mitigation for any resource type where GCP does not expose a checkable limit.

### QuotaProvider interface

GCP-specific quota logic is abstracted behind a `QuotaProvider` interface:

```go
type QuotaProvider interface {
    ListQuotas(ctx context.Context, tenantID, env string) ([]QuotaEntry, error)
    CheckPreProvision(ctx context.Context, tenantID, env, resourceType string) error
}
```

Adding a new cloud provider means implementing this interface. The API layer does not
change.

---

## UCP Platform Soft Quota Layer (future)

The platform soft quota enforces UCP-level per-tenant resource count limits,
independently of GCP. It is the correct mitigation for cases where a cloud provider
does not expose a programmatic quota metric (Cloud SQL instance count being a confirmed
example), and the foundation for consistent quota enforcement across all providers and
resource types.

The design intent:
- A `quota_policies` table stores per-tenant per-resource-type limits
- A `CheckQuota` middleware counts current tenant resources via K8s label-filtered
  queries and blocks `POST` requests that would exceed the limit
- Quota management endpoints allow platform-admins to set and view limits per tenant

Open design questions for this layer are in `poc-report.md`.

---

## Open Questions

1. **Who sets initial platform quotas?** Platform-admin via API, or seeded from some
   external source of capacity intent (e.g. tenant subscription tier)? Needs decision.

2. **Quota inheritance from GCP org quota policies?** If a GCP org policy caps a tenant
   project at 50 CPUs, should UCP read this and reflect it as the tenant's platform
   quota automatically?

3. **Cross-provider quota** — single limit across all providers (e.g. 5 databases total
   regardless of GCP or Omnia), or per-provider limits?

4. **Quota increase workflow** — should tenants be able to request a platform quota
   increase that triggers an approval workflow, optionally followed by a GCP
   `QuotaPreference` submission?

5. **Platform quota drift** — if a resource is deleted directly in GCP outside UCP,
   the platform ledger count is wrong. How frequently should it reconcile against
   actual GCP state? (Connects to the drift detection work.)
