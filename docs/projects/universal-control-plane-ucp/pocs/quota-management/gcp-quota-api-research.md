---
title: "GCP Quota API Research"
space: UCP
parent_page_id: "../quota-management.md"
---

# GCP Quota API Research

**Purpose:** Document the actual GCP APIs for reading quota limits and usage,
with confirmed field structures, before writing any implementation code.

**Verification method:**
- `go doc` on installed genproto packages (proto field definitions)
- `pkg.go.dev/google.golang.org/api/serviceusage/v1beta1` (REST JSON tags)
- `pkg.go.dev/cloud.google.com/go/cloudquotas/apiv1/cloudquotaspb` (Cloud Quotas proto)
- Live API calls against `sub-gcp-ucp-clsd-sandbox` using the cluster credential secret

---

## Cloud Monitoring Quota Metrics — CONFIRMED by live API call

**API:** `GET https://monitoring.googleapis.com/v3/projects/{project}/metricDescriptors`

All metrics below confirmed via `metricDescriptors.list` with filter
`metric.type = starts_with("serviceruntime.googleapis.com/quota")`.

| Metric type | Display name | Kind | Value type | Launch stage |
|-------------|-------------|------|------------|--------------|
| `serviceruntime.googleapis.com/quota/allocation/usage` | Allocation quota usage | GAUGE | INT64 | **GA** |
| `serviceruntime.googleapis.com/quota/limit` | Quota limit | GAUGE | INT64 | **GA** |
| `serviceruntime.googleapis.com/quota/rate/net_usage` | Rate quota usage | DELTA | INT64 | **GA** |
| `serviceruntime.googleapis.com/quota/exceeded` | Quota exceeded error | GAUGE | BOOL | GA |
| `serviceruntime.googleapis.com/quota/concurrent/usage` | Concurrent quota usage | GAUGE | INT64 | ALPHA |
| `serviceruntime.googleapis.com/quota/concurrent/limit` | Concurrent quota limit | GAUGE | INT64 | ALPHA |
| `serviceruntime.googleapis.com/quota/concurrent/exceeded` | Concurrent quota exceeded | DELTA | INT64 | ALPHA |
| ~~`serviceruntime.googleapis.com/quota/allocation/limit`~~ | | | | **DOES NOT EXIST** — HTTP 404 |

---

### `quota/allocation/usage`

Description: *"The total consumed allocation quota. Values reported more than 1/min are dropped."*
Sample period: `60s`

**Labels:**

| Label | Location in response | Example value |
|-------|---------------------|---------------|
| `quota_metric` | `metric.labels` | `compute.googleapis.com/cpus` |
| `location` | `resource.labels` | `us-central1`, `global` |
| `service` | `resource.labels` | `compute.googleapis.com` |
| `project_id` | `resource.labels` | `sub-gcp-ucp-clsd-sandbox` |

Monitored resource type: `consumer_quota`

**Confirmed live data sample** (compute.googleapis.com, asia-east1) — selected non-zero metrics:
```
quota_metric=compute.googleapis.com/cpus                    value=3
quota_metric=compute.googleapis.com/instances               value=3
quota_metric=compute.googleapis.com/instance_groups         value=3
quota_metric=compute.googleapis.com/instance_group_managers value=3
quota_metric=compute.googleapis.com/internal_addresses      value=1
```

Matches GCP Console: 3 GKE node VMs running in asia-east1 (asia-east1-a/b/c).
Metrics with 0 values (`a2_cpus`, `autoscalers`, etc.) are genuinely unused, not missing data.

---

### `quota/limit`

Description: *"The limit for the quota."*
Sample period: `86400s` (daily)

**Labels:**

| Label | Location in response | Example value |
|-------|---------------------|---------------|
| `quota_metric` | `metric.labels` | `compute.googleapis.com/cpus` |
| `limit_name` | `metric.labels` | `CPUS-per-project-region`, `RequestsPerMinutePerRegion` |
| `location` | `resource.labels` | `us-central1`, `global` |
| `service` | `resource.labels` | `compute.googleapis.com` |

Monitored resource type: `consumer_quota`

**Confirmed live data sample** (compute.googleapis.com, asia-east1) — selected values:
```
quota_metric=compute.googleapis.com/cpus                  value=5000
quota_metric=compute.googleapis.com/instances             value=24000
quota_metric=compute.googleapis.com/n2_cpus               value=3000
quota_metric=compute.googleapis.com/autoscalers           value=1250
quota_metric=compute.googleapis.com/instance_groups       value=2500
quota_metric=compute.googleapis.com/list_requests_per_region  value=4500
```

---

### Joining `quota/limit` and `quota/allocation/usage`

Both share these keys: `resource.labels.service`, `resource.labels.location`, `metric.labels.quota_metric`.

`quota/limit` has an extra `metric.labels.limit_name` that `quota/allocation/usage` does not.
If multiple `limit_name` rows exist for the same `(quota_metric, location)`, take the minimum
(most restrictive limit).

---

### Recommended aggregation for "current value" query

```
interval.startTime                = now - 25h
interval.endTime                  = now
aggregation.alignmentPeriod       = 86400s
aggregation.perSeriesAligner      = ALIGN_NEXT_OLDER
```

---

### IAM and OAuth

**IAM permission:** `monitoring.timeSeries.list` — granted by `roles/monitoring.viewer`
**OAuth scope:** `https://www.googleapis.com/auth/cloud-platform` — already in use

---

## Service Usage API v1beta1 — Limits only (alternative)

**Endpoint:**
```
GET https://serviceusage.googleapis.com/v1beta1/projects/{projectId}/services/{service}/consumerQuotaMetrics
```

**Query params:** `pageSize`, `pageToken`

**IAM:** `serviceusage.quotas.get` — `roles/serviceusage.serviceUsageViewer`
Note: **NOT** granted by `roles/cloudquotas.quotasViewer` — different API, different role.

**OAuth scope:** `cloud-platform` (already in use)

### Response structure (confirmed from genproto source)

```
metrics[]
  metric           string          "compute.googleapis.com/cpus"
  displayName      string          "CPUs"
  consumerQuotaLimits[]
    unit           string          "1/{project}/{region}"
    quotaBuckets[]
      effectiveLimit  int64        effective limit — equal to defaultLimit if no override
                                   *** serialized as STRING in REST JSON ***
                                   e.g. "effectiveLimit": "1000"
      defaultLimit    int64        default before any override (also a string in REST JSON)
      dimensions      map[string]string   {"region": "us-central1"}  or  {} for global
nextPageToken  string
```

**No `currentUsage` field** — this API gives limits only.

---

## Cloud Quotas API v1 — Limits + Display Names (original approach, had a bug)

**Endpoint:**
```
GET https://cloudquotas.googleapis.com/v1/projects/{project}/locations/global/services/{service}/quotaInfos
```

**IAM:** `cloudquotas.quotas.get` — `roles/cloudquotas.quotasViewer` — **already added**

### Response structure (confirmed from cloudquotaspb proto)

```
quotaInfos[]
  metric              string          "compute.googleapis.com/cpus"
  service             string          "compute.googleapis.com"
  quotaDisplayName    string          "CPUS per project per region"
  isFixed             bool
  isPrecise           bool
  dimensionsInfos[]                   ← JSON key: "dimensionsInfos" (note the 's')
    dimensions        map[string]string   {"region": "us-central1"}
    details
      value           int64           EFFECTIVE limit (enforced value, including defaults)
    applicableLocations []string
nextPageToken  string
```

**No usage data** — limits and metadata only.

### The original code bug

The original `quota_handler.go` used `json:"dimensionInfos"`.
The correct JSON key is **`dimensionsInfos`** (extra 's' before Infos).
Confirmed by getter method `GetDimensionsInfos()` in the cloudquotaspb package.

This one-character mismatch caused all 495 `dimensionsInfos` arrays to parse as empty
→ every limit showed as 0.

---

## Summary table

| Data needed | Best API | Confirmed real values? | New IAM role needed? |
|-------------|----------|----------------------|----------------------|
| Allocation quota usage | Cloud Monitoring `quota/allocation/usage` | YES — live call | None (already present) |
| Quota limits | Cloud Monitoring `quota/limit` | YES — live call, non-zero values | None (already present) |
| Rate quota usage | Cloud Monitoring `quota/rate/net_usage` | YES — descriptor confirmed | None (already present) |
| Limits (alternative) | Service Usage API v1beta1 `effectiveLimit` | YES — proto confirmed | `serviceusage.serviceUsageViewer` |
| Limits (alternative) | Cloud Quotas API `dimensionsInfos[].details.value` | YES — proto confirmed, fix JSON tag | already have it |

---

## Implementation options

### Option 1 — Cloud Monitoring only (recommended)

One API for both limits and usage:
- **Limits:** `serviceruntime.googleapis.com/quota/limit`
- **Usage:** `serviceruntime.googleapis.com/quota/allocation/usage`
- Join on `(service, quota_metric, location)`

Pros: Single API, real limits AND real usage confirmed, no extra dependencies.
Cons: none — IAM already confirmed present.

### Option 2 — Service Usage API (limits) + Cloud Monitoring (usage)

- Service Usage API v1beta1 `effectiveLimit` for limits + `displayName`
- Cloud Monitoring `quota/allocation/usage` for usage

Pros: Gets display names directly.
Cons: Two APIs, needs `serviceusage.serviceUsageViewer` (not confirmed present).

### Option 3 — Fix Cloud Quotas API JSON tag bug (limits only, no usage)

- Change `json:"dimensionInfos"` → `json:"dimensionsInfos"` in original code
- Already authorized, no new IAM roles
- Usage stays 0

Pros: One-line fix, no new roles.
Cons: No usage data.

---

## IAM roles status

| Role | Permission | API | Status |
|------|------------|-----|--------|
| `roles/cloudquotas.quotasViewer` | `cloudquotas.quotas.get` | Cloud Quotas API | **Already added** |
| `roles/monitoring.viewer` | `monitoring.timeSeries.list` | Cloud Monitoring | **Already present** (confirmed by live API calls) |
| `roles/serviceusage.serviceUsageViewer` | `serviceusage.quotas.get` | Service Usage API | **Unknown — only needed for Option 2** |
