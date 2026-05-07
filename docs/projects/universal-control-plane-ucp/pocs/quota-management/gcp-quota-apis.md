---
title: "GCP Quota APIs"
space: UCP
parent_page_id: "../quota-management.md"
---

# GCP Quota APIs

## API Overview

Two GCP APIs manage quotas programmatically:

| API | Purpose | Status |
|---|---|---|
| `cloudquotas.googleapis.com` | View quota metadata, submit increase requests | GA since Q1 2024 |
| `serviceusage.googleapis.com` | List quotas, set consumer overrides (cap/reduce) | GA (older design) |
| `monitoring.googleapis.com` | Read real-time quota usage metrics | GA |

These serve different use cases and are complementary, not interchangeable.

---

## Cloud Quotas API (`cloudquotas.googleapis.com`)

The dedicated, first-class API for programmatic quota management. Reached GA in Q1 2024.

**Key capabilities:**

- Browse quota metadata (`QuotaInfo` — dimensions, units, current effective limit)
- Submit quota increase requests (`QuotaPreference`)
- Track approval state of increase requests
- Apply quota caps at org/folder level (enterprise)

**What it CANNOT do:**

- Set consumer overrides (reduce quota below default) — that is ServiceUsage API
- Return real-time usage data — that is Cloud Monitoring

### Core Resources

**QuotaInfo** (read-only): metadata about a quota — its dimensions, current effective
limit, default value. Does not include real-time usage.

```
GET https://cloudquotas.googleapis.com/v1/projects/{project}/locations/global/quotaInfos
GET https://cloudquotas.googleapis.com/v1/projects/{project}/locations/global/quotaInfos/{quotaId}
```

**QuotaPreference** (read-write): an adjustment request — your desired quota value
submitted to GCP's quota approval pipeline.

```
POST   https://cloudquotas.googleapis.com/v1/projects/{project}/locations/global/quotaPreferences
PATCH  https://cloudquotas.googleapis.com/v1/projects/{project}/locations/global/quotaPreferences/{id}
GET    https://cloudquotas.googleapis.com/v1/projects/{project}/locations/global/quotaPreferences/{id}
GET    https://cloudquotas.googleapis.com/v1/projects/{project}/locations/global/quotaPreferences
```

### QuotaPreference Structure

```json
{
  "name": "projects/123/locations/global/quotaPreferences/compute.googleapis.com%2Fcpus-us-central1",
  "service": "compute.googleapis.com",
  "quotaId": "CPUS-per-project-region",
  "quotaConfig": {
    "preferredValue": "200"
  },
  "dimensions": {
    "region": "us-central1"
  },
  "justification": "Running large ML training jobs requires 200 CPUs.",
  "contactEmail": "owner@example.com",
  "state": "APPROVED"
}
```

State lifecycle: `PENDING` → `APPROVED` / `DENIED` / `CANCELLED`

Small increases on widely available resources are often auto-approved. GPU quotas and
large increases (10x+) require manual review and can take 2–3 business days or more.

**Important:** The `quotaId` (e.g. `CPUS-per-project-region`) is NOT the same as the
Cloud Monitoring metric name (e.g. `compute.googleapis.com/quota/cpus/limit`). Get the
`quotaId` from `quotaInfos.list` or the GCP Console quota page.

### Rate Limits

- `quotaPreferences.create/patch`: 1 request/second per project
- `quotaInfos.list/get`, `quotaPreferences.list/get`: 60 requests/minute per project

---

## ServiceUsage API (`serviceusage.googleapis.com`)

The older API, originally for enabling/disabling GCP services. Quota support was added
as a secondary concern. Still fully supported.

**Key capabilities:**

- List all quota metrics and current effective limits on a project
- Set **consumer overrides** — reduce or pin a quota below the system default
- Set **admin overrides** — grant additional quota (requires `servicemanagement.admin`)

**What it CANNOT do:**

- Submit quota increase requests (only Cloud Quotas API can do this)
- Return real-time usage (only Cloud Monitoring)
- Increase quota above the system default via consumer overrides — consumer overrides
  can only lower/pin quota, not raise it

### Key Operations

| Operation | Endpoint |
|---|---|
| List quota metrics | `GET .../services/{service}/consumerQuotaMetrics` |
| Get specific metric | `GET .../services/{service}/consumerQuotaMetrics/{metric}` |
| List consumer overrides | `GET .../limits/{limit}/consumerOverrides` |
| Create consumer override | `POST .../limits/{limit}/consumerOverrides` |
| Update consumer override | `PATCH .../limits/{limit}/consumerOverrides/{id}` |
| Delete consumer override | `DELETE .../limits/{limit}/consumerOverrides/{id}` |

**Consumer override use case:** Enforce a lower limit on a sandbox or test project.
For example, if GCP grants a project 1,000 CPUs, the platform admin can set a consumer
override of 100 to prevent runaway usage. This is immediate and programmatic.

### Required IAM Permissions

- `serviceusage.quotas.get` — read quotas
- `serviceusage.quotas.update` — create/update consumer overrides
- Predefined role: `roles/serviceusage.serviceUsageConsumer` or `roles/editor`

### Rate Limits

- Quota read operations: 600 requests/minute per project
- Quota write operations (overrides): 60 requests/minute per project

---

## API Capability Comparison

| Operation | Cloud Quotas API | ServiceUsage API |
|---|---|---|
| List quota metadata | `quotaInfos.list` | `consumerQuotaMetrics.list` |
| Get current limit | `quotaInfos.get` | `consumerQuotaMetrics.get` |
| Get current **usage** | — | — (use Cloud Monitoring) |
| Request quota **increase** | `quotaPreferences.create` | — |
| Set consumer override (cap) | — | `consumerOverrides.create` |
| Set admin override (grant) | — | `adminOverrides.create` (restricted) |
| Delete override | — | `consumerOverrides.delete` |

---

## Cloud Monitoring — Quota Usage Metrics

Real-time quota usage is available only via Cloud Monitoring, not via the quota APIs.

### Metric Name Patterns

```
{service}/quota/{resource}/usage     — current consumption
{service}/quota/{resource}/limit     — current effective limit
{service}/quota/{resource}/exceeded  — count of quota exceeded events
```

### Compute Engine Examples

```
compute.googleapis.com/quota/cpus/usage
compute.googleapis.com/quota/cpus/limit
compute.googleapis.com/quota/in_use_addresses/usage
compute.googleapis.com/quota/disks_total_gb/usage
compute.googleapis.com/quota/nvidia_p100_gpus/usage
```

### Cloud SQL

```
sqladmin.googleapis.com/quota/instances/usage
sqladmin.googleapis.com/quota/instances/limit
```

### Querying via gcloud

```bash
gcloud monitoring time-series list \
  --filter='metric.type="compute.googleapis.com/quota/cpus/usage"' \
  --project=PROJECT

# Per-region Compute quota with usage (simplest one-liner)
gcloud compute regions describe us-central1 --project=PROJECT \
  --format="table(quotas.metric, quotas.limit, quotas.usage)"
```

### Quota Alerts via Cloud Monitoring

Cloud Monitoring alerting policies can fire on quota usage percentage. There is no
native `usage_percentage` metric — you compute it via MQL:

```
# Alert when CPU usage exceeds 80% of limit in us-central1
fetch consumer_quota
| metric 'compute.googleapis.com/quota/cpus/usage'
| filter resource.region == 'us-central1'
| ratio_to_percent
    fetch consumer_quota
    | metric 'compute.googleapis.com/quota/cpus/limit'
    | filter resource.region == 'us-central1'
| condition val() > 80
```

GCP also offers a shortcut: from the Quotas console page, clicking "Set Alert" on any
quota row auto-creates a Monitoring alerting policy for that quota.

**Recommended threshold:** Alert at 70–80% to allow time to request an increase before
operations start failing at 100%.

---

## gcloud CLI Commands

### Cloud Quotas API (newer, `gcloud alpha quotas`)

```bash
# List quota metadata for a service
gcloud alpha quotas info list \
  --project=PROJECT --service=compute.googleapis.com

# Get details for a specific quota
gcloud alpha quotas info describe CPUS-per-project-region \
  --project=PROJECT --service=compute.googleapis.com

# Submit a quota increase request
gcloud alpha quotas preferences create \
  --project=PROJECT \
  --service=compute.googleapis.com \
  --quota-id=CPUS-per-project-region \
  --preferred-value=200 \
  --dimensions="region=us-central1" \
  --justification="Need more CPUs for ML training" \
  --email=owner@example.com

# Check status of increase requests
gcloud alpha quotas preferences list --project=PROJECT
gcloud alpha quotas preferences describe PREFERENCE_ID --project=PROJECT
```

Note: `gcloud alpha quotas` targets the Cloud Quotas API. These remain under `alpha`
in gcloud even though the API itself is GA.

### ServiceUsage API (older, `gcloud alpha services quota`)

```bash
# List quota metrics
gcloud alpha services quota list \
  --service=compute.googleapis.com --project=PROJECT

# Set a consumer override (reduce/pin quota)
gcloud alpha services quota update compute.googleapis.com/cpus \
  --service=compute.googleapis.com \
  --project=PROJECT \
  --value=50 \
  --dimensions="region=us-central1"
```

---

## Terraform

### `google_service_usage_consumer_quota_override`

Manages **consumer overrides** (reduce quota below GCP-granted limit) via the
ServiceUsage API.

```hcl
resource "google_service_usage_consumer_quota_override" "cpu_cap" {
  project        = "my-project-id"
  service        = "compute.googleapis.com"
  metric         = "compute.googleapis.com%2Fcpus"    # URL-encoded
  limit          = "%2Fproject%2Fregion"              # URL-encoded
  override_value = "50"
  dimensions = {
    region = "us-central1"
  }
  force = true  # required if override reduces below current usage
}
```

**Important limitation:** This resource can only set values at or below the system
default. It cannot request quota increases above what GCP has granted.

As of January 2025, there is **no Terraform resource for `QuotaPreference`** (quota
increase requests). Quota increase requests cannot be fully managed via Terraform state.

---

## Crossplane — Gap

**No dedicated quota management CRDs exist in provider-upjet-gcp as of January 2025.**

The provider covers infrastructure provisioning (VMs, clusters, databases) but not quota
management. There is no `serviceusage` or `cloudquotas` service family in the provider.

The `google_service_usage_consumer_quota_override` resource exists in the Terraform
provider but was not included in the provider-upjet-gcp Upjet code generation.

### Workarounds if Crossplane-based quota management is needed

1. **provider-terraform** — use a `Workspace` managed resource that executes
   `google_service_usage_consumer_quota_override` HCL
2. **Custom controller** — write a Go controller that calls the Cloud Quotas API
   directly and is managed as a Crossplane MR
3. **Composition Function** — invoke the Cloud Quotas REST API via HTTP in a
   Function pipeline step

None of these are native. This is a gap in the Crossplane ecosystem for quota management.
