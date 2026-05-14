---
title: "GCP Quota API Reference"
space: UCP
parent_page_id: "../quota-management.md"
---

# GCP Quota API Reference

Three GCP APIs are relevant to quota management. Only Cloud Monitoring is used in the
current implementation. The others are documented here as reference.

| API | Used in UCP | Purpose |
|-----|------------|---------|
| `monitoring.googleapis.com` | **Yes — primary** | Real-time limit and usage metrics |
| `cloudquotas.googleapis.com` | No | Quota metadata, increase requests |
| `serviceusage.googleapis.com` | No | Effective limits, consumer overrides |

---

## Cloud Monitoring API — Primary (confirmed by live call)

**Endpoint:** `GET https://monitoring.googleapis.com/v3/projects/{project}/timeSeries`

**IAM:** `roles/monitoring.viewer` (`monitoring.timeSeries.list`) — already present on
the UCP service accounts (confirmed by successful live API calls).

**OAuth scope:** `https://www.googleapis.com/auth/cloud-platform` — already in use.

### Confirmed quota metrics

All confirmed via `metricDescriptors.list` with filter
`metric.type = starts_with("serviceruntime.googleapis.com/quota")` against the sandbox
project `sub-gcp-ucp-clsd-sandbox`.

| Metric type | Kind | Value type | GA | Notes |
|-------------|------|------------|----|-------|
| `serviceruntime.googleapis.com/quota/limit` | GAUGE | INT64 | ✅ | Effective limit per (metric, location) |
| `serviceruntime.googleapis.com/quota/allocation/usage` | GAUGE | INT64 | ✅ | Current allocation usage |
| `serviceruntime.googleapis.com/quota/rate/net_usage` | DELTA | INT64 | ✅ | Rate quota usage |
| `serviceruntime.googleapis.com/quota/exceeded` | GAUGE | BOOL | ✅ | Quota exceeded flag |
| `serviceruntime.googleapis.com/quota/concurrent/usage` | GAUGE | INT64 | ALPHA | — |
| `serviceruntime.googleapis.com/quota/concurrent/limit` | GAUGE | INT64 | ALPHA | — |
| ~~`serviceruntime.googleapis.com/quota/allocation/limit`~~ | — | — | ❌ | **Does not exist — HTTP 404** |

> **Important:** The metric path is `serviceruntime.googleapis.com/quota/...` — NOT
> `{service}/quota/{resource}/...`. The latter format (e.g.
> `compute.googleapis.com/quota/cpus/usage`) is documentation shorthand and does not work
> as an actual Cloud Monitoring filter.

### `quota/limit` — confirmed live data (compute.googleapis.com, asia-east1)

```
quota_metric=compute.googleapis.com/cpus                  limit=5000
quota_metric=compute.googleapis.com/instances             limit=24000
quota_metric=compute.googleapis.com/n2_cpus               limit=3000
quota_metric=compute.googleapis.com/autoscalers           limit=1250
quota_metric=compute.googleapis.com/instance_groups       limit=2500
```

Labels:
- `metric.labels.quota_metric` — e.g. `compute.googleapis.com/cpus`
- `metric.labels.limit_name` — e.g. `CPUS-per-project-region` (**only on `quota/limit`**)
- `resource.labels.location` — e.g. `asia-east1`, `global`
- `resource.labels.service` — e.g. `compute.googleapis.com`

### `quota/allocation/usage` — confirmed live data (compute.googleapis.com, asia-east1)

```
quota_metric=compute.googleapis.com/cpus                    usage=3
quota_metric=compute.googleapis.com/instances               usage=3
quota_metric=compute.googleapis.com/instance_groups         usage=3
quota_metric=compute.googleapis.com/instance_group_managers usage=3
quota_metric=compute.googleapis.com/internal_addresses      usage=1
```

Matches GCP Console: 3 GKE node VMs in asia-east1. Zero-valued metrics (e.g. `a2_cpus`,
`autoscalers`) are genuinely unused, not missing data.

Labels: same as `quota/limit` **except no `limit_name`**.

### Joining limit and usage

Both metrics share: `(resource.labels.service, resource.labels.location, metric.labels.quota_metric)`.

`quota/limit` has an extra `metric.labels.limit_name` — multiple rows can exist for the
same `(quota_metric, location)` key. Take the **minimum** to surface the most restrictive
limit.

### Recommended query parameters

```
interval.startTime              = now - 25h  (RFC3339)
interval.endTime                = now         (RFC3339)
aggregation.alignmentPeriod     = 86400s
aggregation.perSeriesAligner    = ALIGN_NEXT_OLDER
pageSize                        = 1000
```

This returns one data point per series (the most recent value). The 25h window ensures
data is available even if the metric was last reported up to a day ago.

### INT64_MAX sentinel

GCP uses `9223372036854775807` (INT64_MAX) for "no cap" quotas. Filter these out with:

```go
const unlimitedThreshold int64 = 1_000_000_000_000_000  // 10^15
limitAvailable = limit > 0 && limit < unlimitedThreshold
```

### Cloud SQL note

`serviceruntime.googleapis.com/quota/allocation/usage` does **not** return Cloud SQL
instance count. Cloud SQL's monitored quotas are rate quotas (Admin API calls/min), not
resource allocation counts. The Cloud SQL instance limit is a soft limit with no
programmatic API support.

---

## Cloud Quotas API (`cloudquotas.googleapis.com`)

GA since Q1 2024. Provides quota metadata and increase requests. **Not used in current
UCP implementation** due to JSON tag bug (see below) and because Cloud Monitoring provides
both limits and usage in one API.

**IAM:** `roles/cloudquotas.quotasViewer` — already added to UCP service accounts.

**Endpoints:**

```
GET  /v1/projects/{project}/locations/global/quotaInfos
GET  /v1/projects/{project}/locations/global/quotaInfos/{quotaId}
POST /v1/projects/{project}/locations/global/quotaPreferences
```

**Response structure (confirmed from `cloudquotaspb` proto):**

```
quotaInfos[]
  metric              string    "compute.googleapis.com/cpus"
  service             string    "compute.googleapis.com"
  quotaDisplayName    string    "CPUS per project per region"
  isFixed             bool
  dimensionsInfos[]             ← JSON key is "dimensionsInfos" (note the 's')
    dimensions        map       {"region": "us-central1"}
    details
      value           int64     effective limit
nextPageToken  string
```

**Known bug (original UCP code):** The original `quota_handler.go` used
`json:"dimensionInfos"`. The correct JSON key is `json:"dimensionsInfos"` (extra `s`
before `Infos`). This caused all 495 `dimensionsInfos` arrays to parse as empty — every
limit showed 0. The rewrite to Cloud Monitoring eliminated this dependency.

**QuotaPreference** (increase requests):

```json
{
  "service": "compute.googleapis.com",
  "quotaId": "CPUS-per-project-region",
  "quotaConfig": { "preferredValue": "200" },
  "dimensions": { "region": "us-central1" },
  "justification": "...",
  "state": "PENDING"
}
```

State lifecycle: `PENDING` → `APPROVED` / `DENIED` / `CANCELLED`. Small increases are
often auto-approved; GPU/large increases require manual review (2–3 business days+).

---

## ServiceUsage API (`serviceusage.googleapis.com`)

**Not used in current UCP implementation.** Requires `roles/serviceusage.serviceUsageViewer`
which has not been confirmed on UCP service accounts.

Primary use case: **consumer overrides** — reducing quota below the GCP-granted default.
Useful for enforcing caps on sandbox projects.

```
GET  .../services/{service}/consumerQuotaMetrics
POST .../limits/{limit}/consumerOverrides   ← cap quota below default
```

**Response structure (`effectiveLimit` confirmed from genproto source):**

```
metrics[]
  metric           string    "compute.googleapis.com/cpus"
  displayName      string    "CPUs"
  consumerQuotaLimits[]
    quotaBuckets[]
      effectiveLimit  int64  (serialized as STRING in REST JSON: "effectiveLimit": "1000")
      dimensions      map    {"region": "us-central1"}
```

Note: `effectiveLimit` is serialized as a JSON string, not a number.

**Limitation:** Cannot return real-time usage. Does not support quota increase requests
(only Cloud Quotas API can do that).

---

## gcloud CLI reference

```bash
# Cloud Quotas API
gcloud alpha quotas info list --project=PROJECT --service=compute.googleapis.com
gcloud alpha quotas preferences create \
  --project=PROJECT --service=compute.googleapis.com \
  --quota-id=CPUS-per-project-region --preferred-value=200 \
  --dimensions="region=us-central1" --justification="..."

# Cloud Monitoring — quota usage
gcloud monitoring time-series list \
  --filter='metric.type="serviceruntime.googleapis.com/quota/allocation/usage" AND resource.labels.service="compute.googleapis.com"' \
  --project=PROJECT

# Compute quota shortcut (regional)
gcloud compute regions describe us-central1 --project=PROJECT \
  --format="table(quotas.metric, quotas.limit, quotas.usage)"
```

---

## Crossplane gap

No quota management CRDs exist in `provider-upjet-gcp`. The `google_service_usage_consumer_quota_override`
Terraform resource exists but was not included in Upjet code generation. Workarounds if
Crossplane-based quota management is needed:

1. **provider-terraform** — `Workspace` MR executing `google_service_usage_consumer_quota_override`
2. **Custom controller** — Go controller calling Cloud Quotas API directly
3. **Composition Function** — HTTP call to Cloud Quotas REST API in a Function pipeline step
