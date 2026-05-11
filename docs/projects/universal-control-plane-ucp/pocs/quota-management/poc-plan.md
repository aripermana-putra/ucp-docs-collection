---
title: "Quota Management PoC Plan"
space: UCP
parent_page_id: "../quota-management.md"
---

# Quota Management PoC Plan

## Goal

Prove that UCP can read real-time quota usage and limits from a cloud provider (GCP first)
using the tenant's existing credentials, surface those quotas through a UCP API endpoint,
and block resource provisioning at the UCP API layer before a quota-exceeded request ever
reaches the cloud provider.

This PoC does NOT implement UCP-level platform soft quotas (the `quota_policies` table and
`CheckQuota` middleware from `docs/architecture/RBAC.md` Phase 2). That is a separate
concern. This PoC is about communicating with the cloud provider's own quota system.

---

## Scope

### In Scope

- **Quota listing endpoint** — `GET /api/v1/quota`: returns all relevant quota metrics
  (usage + limit) for the calling tenant's GCP project, sourced from GCP APIs.
- **Pre-provision gate** — `POST /api/v1/databases`: before applying the XR, check the
  tenant's Cloud SQL instance quota against current usage. Return a clear error (HTTP 429)
  if the quota is reached. The XR is never created.
- **Provider interface** — define a `QuotaProvider` interface in the API server so the
  GCP implementation is one concrete implementation, not hardcoded logic. This is the
  extensibility contract for future providers.
- **Resource type**: Cloud SQL instances only.

### Out of Scope

- UCP platform-level soft quotas (`quota_policies` table, `CheckQuota` middleware).
- Quota increase requests (Cloud Quotas API `QuotaPreference`) — triggering increases is
  not part of this PoC.
- Other resource types beyond Cloud SQL instances.
- Other cloud providers (AWS, Azure, Omnia) — covered by the provider checklist below.
- Frontend quota UI.
- Quota alerting or threshold notifications.

---

## Technical Approach

### Quota Listing

GCP exposes quota data across two separate APIs:

| Data | API | Method |
|---|---|---|
| Current usage | Cloud Monitoring API | Query `sqladmin.googleapis.com/quota/instances/usage` metric |
| Current limit | Cloud Quotas API | `quotaInfos.get` for `cloudsql.googleapis.com` service |

The endpoint aggregates both to return `{metric, usage, limit}` rows.

**Credentials:** reuse the GCP service account JSON already stored per tenant from the
credential upload flow. No additional credentials are needed.

### Pre-Provision Gate

In the `POST /api/v1/databases` handler, before calling the Temporal workflow or applying
the XR:

1. Resolve the tenant's GCP project from their `ProviderConfig`.
2. Call `QuotaProvider.CheckPreProvision(ctx, tenantID, env, "database")`.
3. The GCP implementation queries Cloud Monitoring for current Cloud SQL instance count
   and Cloud Quotas API for the limit.
4. If `usage >= limit`, return HTTP 429 with a descriptive error message.
5. If quota is available, proceed with provisioning as normal.

### Provider Interface

```go
type QuotaEntry struct {
    Metric  string // e.g. "sqladmin.googleapis.com/quota/instances"
    Usage   int64
    Limit   int64
}

type QuotaProvider interface {
    ListQuotas(ctx context.Context, tenantID, env string) ([]QuotaEntry, error)
    CheckPreProvision(ctx context.Context, tenantID, env, resourceType string) error
}
```

`resourceType` maps to provider-specific quota metrics via a static mapping in each
implementation:

| UCP resource type | GCP quota metric |
|---|---|
| `database` | `sqladmin.googleapis.com/quota/instances` |
| `compute` | `compute.googleapis.com/quota/cpus` (regional — out of PoC scope) |
| `k8s-cluster` | `container.googleapis.com/quota/clusters` (out of PoC scope) |

---

## Provider-Agnostic Checklist

When implementing quota management for a new cloud provider, answer every item in this
checklist. If any item cannot be answered, quota management for that provider is blocked.

### API Availability

- [ ] Does the provider expose a quota **listing** API (returns quota names, limits)?
- [ ] Does the provider expose **real-time usage** data via a metrics or monitoring API?
- [ ] Are both APIs stable and production-ready (GA or equivalent)?
- [ ] What are the rate limits on the quota APIs?

### Credentials

- [ ] Are the quota APIs accessible with the same credentials used for provisioning?
- [ ] If separate credentials are required, what permissions/roles are needed?
- [ ] Is there a read-only role that grants quota visibility without provisioning access?

### Resource Type Mapping

- [ ] What is the exact quota metric name for each UCP resource type?
  - `database` → ?
  - `compute` → ?
  - `k8s-cluster` → ?
  - `storage` → ?
  - `terraform` → ? (provider-specific)
- [ ] Are quotas scoped globally or per-region? If regional, how is the target region
  resolved at pre-provision check time?

### Error Handling

- [ ] What HTTP status and error structure does the provider return when quota is exceeded
  during provisioning?
- [ ] Is there a structured error field to distinguish quota-exceeded from other errors
  (e.g. `reason: quotaExceeded` vs `reason: rateLimitExceeded`)?

### Tenant Project Mapping

- [ ] How is the tenant mapped to the cloud account/project that holds the quota?
  (For GCP: via `ProviderConfig` name → GCP project ID embedded in service account JSON.)

### Increase Workflow (Future)

- [ ] Does the provider support programmatic quota increase requests?
- [ ] Is the increase synchronous (immediate) or asynchronous (approval pipeline)?
- [ ] What is the typical approval lead time?

---

## Success Criteria

The PoC is considered complete when:

1. `GET /api/v1/quota` returns real Cloud SQL quota usage and limit for a tenant's GCP
   project, using their stored service account credentials.
2. When a tenant's Cloud SQL instance count is at the GCP limit, `POST /api/v1/databases`
   returns HTTP 429 with a clear error message — no XR or Temporal workflow is started.
3. When quota is available, `POST /api/v1/databases` behaves identically to today.
4. The `QuotaProvider` interface exists and the GCP implementation is one concrete struct
   behind it — no GCP-specific logic leaks into the handler.

---

## What This PoC Does Not Decide

- Whether UCP will add its own platform-level soft quotas on top of provider quotas.
- Whether quota management will be integrated into the Crossplane layer (admission webhook
  or composition function) rather than the API server.
- The final shape of the quota UI.
- Cross-provider quota aggregation (e.g. a single limit across GCP + AWS resources).
