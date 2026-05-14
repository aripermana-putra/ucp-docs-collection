---
title: "Quota Management — Implementation"
space: UCP
parent_page_id: "../quota-management.md"
---

# Quota Management — Implementation

## Goal

Prove that UCP can read real-time quota usage and limits from a cloud provider (GCP first)
using the tenant's existing credentials, surface those quotas through a UCP API endpoint,
and block resource provisioning at the UCP API layer before a quota-exceeded request ever
reaches the cloud provider.

This is distinct from UCP platform-level soft quotas (`quota_policies` table, `CheckQuota`
middleware). That is a separate concern. See `ucp-quota-design.md`.

---

## Current Implementation State

| Component | File | Status |
|---|---|---|
| `QuotaProvider` interface | `backend/api-server/quota_handler.go` | **Implemented** |
| `gcpQuotaProvider` struct | `backend/api-server/quota_handler.go` | **Implemented** |
| `GET /api/v1/quota` endpoint | `backend/api-server/main.go:197` | **Deployed** |
| Pre-provision gate | `backend/api-server/main.go:478` | **Implemented (no-op for GCP)** |
| Quota UI — table with filters | `frontend/src/components/QuotaList.jsx` | **Deployed** |

### What was originally planned vs what was built

The original plan used the Cloud Quotas API for limits and Cloud Monitoring for usage.
During implementation, the Cloud Quotas API JSON tag bug (`dimensionInfos` vs
`dimensionsInfos`) caused all limits to return 0. The final implementation switched to
**Cloud Monitoring only** for both limits and usage — confirmed by live API calls.

See `gcp-api-reference.md` for the full API research and confirmed field structures.

---

## Scope

### In Scope (implemented)

- **`GET /api/v1/quota`** — returns all quota metrics (usage + limit + percentage) for the
  tenant's GCP project across four services: Cloud SQL, Compute Engine, GKE, Cloud Storage
- **`QuotaProvider` interface** — abstracts GCP-specific logic; adding a new provider means
  implementing the interface
- **Pre-provision gate** — `CheckPreProvision` wired to `POST /api/v1/databases`; returns
  HTTP 429 before creating any XR or Temporal workflow if quota is exhausted. Currently a
  no-op for GCP (fails open) — see findings below
- **Quota UI** — filterable table in the frontend with service filter, metric search,
  sortable columns, usage % bar, pagination

### Out of Scope

- UCP platform-level soft quotas (`quota_policies`, `CheckQuota` middleware)
- Quota increase requests (`QuotaPreference` API)
- Pre-provision gates for resource types other than Cloud SQL
- Other cloud providers (AWS, Azure, Omnia)
- Quota alerting or threshold notifications

---

## API Sequence

`GET /api/v1/quota?tenantId=<rns>`

```mermaid
sequenceDiagram
    autonumber

    participant Browser as Browser<br/>(QuotaList.jsx)
    participant SessionDB as API Server PostgreSQL<br/>(BFF session store)
    participant Keycloak as Keycloak
    participant Handler as API Server<br/>(ListQuota)
    participant Horizon as Horizon API QA<br/>(qa-horizon-data-api.r-local.net)
    participant K8s as Kubernetes<br/>(crossplane-system)
    participant GCPAuth as GCP OAuth2
    participant Monitoring as Cloud Monitoring API

    Browser->>Handler: GET /api/v1/quota?tenantId=rns:rakuten:ucp:xxx
    Note over Browser,Handler: Cookie: ucp-session=abc123<br/>X-Environment: QA<br/>Authorization: Bearer &lt;user-jwt&gt;

    Note over Handler: SessionMiddleware runs first

    Handler->>SessionDB: GetSessionWithUser("abc123")
    SessionDB-->>Handler: session{enc_access_token, enc_refresh_token, expires_at}<br/>user{email, username, ...}

    alt access token expired
        Handler->>Keycloak: POST /protocol/openid-connect/token<br/>grant_type=refresh_token
        Keycloak-->>Handler: new access_token + refresh_token
        Handler->>SessionDB: UpdateSessionTokens(session_id, new_tokens, extended_expiry)
    end

    Note over Handler: Principal{email, access_token} injected into request context

    Note over Handler: ListQuota handler starts

    Handler->>Handler: tenantID = query["tenantId"] → "rns:rakuten:ucp:xxx"

    Note over Handler: requireTenantAdmin() check

    Handler->>Handler: extractBearerToken(r)<br/>→ reads Principal.AccessToken from context (set by SessionMiddleware)<br/>→ falls back to Authorization header only if context is empty<br/>extractEmailFromJWT(token) → reads "email" claim from JWT payload

    Handler->>Horizon: GET /v0/tenants/rns:rakuten:ucp:xxx
    Note over Handler,Horizon: Authorization: Bearer &lt;user-jwt&gt;
    Horizon-->>Handler: {"admins": [{"email": "user@example.com"}, ...]}
    Handler->>Handler: user email ∈ admins[] → authorized ✓

    Handler->>Handler: getEnvironment(r)<br/>X-Environment: "QA" → env = "qa"

    Note over Handler,K8s: gcpHTTPClient() — resolve GCP credentials

    Handler->>Handler: credentialSecretName("gcp", tenantID, "qa")<br/>→ "cloud-credentials-gcp-&lt;sanitized-rns&gt;-qa"

    Handler->>K8s: GET /api/v1/namespaces/crossplane-system/secrets/<br/>cloud-credentials-gcp-&lt;sanitized-rns&gt;-qa
    K8s-->>Handler: Secret{data: {credentials.json: &lt;b64&gt;, project_id: &lt;b64&gt;}}

    Handler->>Handler: base64-decode credentials.json → GCP service account JSON<br/>base64-decode project_id → "my-gcp-project-qa"

    Handler->>GCPAuth: POST /token (service account JWT bearer)<br/>scope=https://www.googleapis.com/auth/cloud-platform
    Note over Handler,GCPAuth: TLS InsecureSkipVerify=true for corporate proxy
    GCPAuth-->>Handler: {access_token: "ya29.xxx", expires_in: 3600}

    loop for each service in [sqladmin, compute, container, storage] — 8 total calls

        Handler->>Monitoring: GET /v3/projects/my-gcp-project-qa/timeSeries
        Note over Handler,Monitoring: filter=metric.type="serviceruntime.googleapis.com/quota/limit"<br/>  AND resource.labels.service="&lt;service&gt;"<br/>interval.startTime=&lt;now-25h&gt;, interval.endTime=&lt;now&gt;<br/>aggregation.alignmentPeriod=86400s, perSeriesAligner=ALIGN_NEXT_OLDER<br/>pageSize=1000
        Monitoring-->>Handler: timeSeries[]{metric.labels{quota_metric, limit_name},<br/>resource.labels{location}, points[0].value.int64Value}
        Note over Handler: Keep minimum value per (quota_metric, location)<br/>to handle multiple limit_name rows for same key

        Handler->>Monitoring: GET /v3/projects/my-gcp-project-qa/timeSeries
        Note over Handler,Monitoring: filter=metric.type="serviceruntime.googleapis.com/quota/allocation/usage"<br/>  AND resource.labels.service="&lt;service&gt;"
        Monitoring-->>Handler: timeSeries[]{metric.labels{quota_metric},<br/>resource.labels{location}, points[0].value.int64Value}
        Note over Handler: Join on (quota_metric, location)<br/>limitAvailable = limit > 0 AND limit < 10^15<br/>percentage = usage/limit*100 if limitAvailable

    end

    Handler-->>Browser: 200 OK {"count": 842, "quotas": [...]}
    Browser->>Browser: render table: service / metric / region / limit / usage / usage%
```

### Key parameters

**X-Environment routing:**

| Header | `env` | Horizon URL | Credential secret suffix |
|--------|-------|-------------|--------------------------|
| `QA` | `qa` | `https://qa-horizon-data-api.r-local.net/v0` | `-qa` |
| `PRODUCTION` | `production` | `https://horizon-data-api.r-local.net/v0` | `-production` |

**Cloud Monitoring call — 2 × 4 services = 8 calls per request:**

```
GET https://monitoring.googleapis.com/v3/projects/{projectID}/timeSeries
  ?filter=metric.type="{metricType}" AND resource.labels.service="{service}"
  &interval.startTime={now-25h RFC3339}
  &interval.endTime={now RFC3339}
  &aggregation.alignmentPeriod=86400s
  &aggregation.perSeriesAligner=ALIGN_NEXT_OLDER
  &pageSize=1000
```

**Join and sentinel logic:**

```
key             = (metric.labels.quota_metric, resource.labels.location)
limits[key]     = min(all values for that key)   ← quota/limit has limit_name duplicates
usage[key]      = single value

limitAvailable  = limit > 0 AND limit < 1_000_000_000_000_000   ← filters INT64_MAX sentinel
percentage      = float64(usage) / float64(limit) * 100
```

---

## Key Findings

### Finding 1: GCP quota data is far more numerous than expected

A single GCP service exposes dozens of quota metrics, each producing multiple rows across
regions. A typical project returns 800–900 rows total across four services. The GCP
Console quota page was the wrong design reference — it is built for project administrators,
not for UCP tenants making provisioning decisions.

### Finding 2: Cloud SQL instance count is not available via quota API

The quota that matters most for the pre-provision gate (`CheckPreProvision`) — Cloud SQL
instances per project — is a **soft limit** with no metric ID in any quota API. It is
managed via GCP support cases. The `serviceruntime.googleapis.com/quota/allocation/usage`
metric does not return Cloud SQL instance count.

This means the pre-provision gate for databases cannot be implemented as originally
designed. It currently fails open (no error → provisioning proceeds).

### Finding 3: Cloud SQL quota metrics are rate quotas, not resource count quotas

The Cloud SQL metrics that do have metric IDs (`connect`, `get`, `list`, `mutate`) are
**rate quotas** — limits on Admin API call frequency per minute. They are irrelevant to
UCP tenant capacity planning. A tenant managing a handful of databases will never approach
these limits.

### Finding 4: Compute Engine and GKE usage metrics work correctly

`serviceruntime.googleapis.com/quota/allocation/usage` does return real data for Compute
Engine and GKE (confirmed: `cpus=3`, `instances=3` for asia-east1 matching 3 GKE node
VMs). Cloud Monitoring is the correct and sufficient API for these.

### Finding 5: The Cloud Quotas API had a JSON tag bug in the original implementation

The original code used `json:"dimensionInfos"`. The correct key is `json:"dimensionsInfos"`
(extra `s`). This caused all 495 quota entries to parse as empty — every limit showed 0.
The rewrite switched to Cloud Monitoring only, eliminating this dependency.

---

## Display Strategy

Three options were considered for what to show in the quota table:

| Option | Approach | Decision |
|--------|----------|----------|
| A — Mirror GCP Console | Show every metric returned by API | Too complex, GCP-specific |
| B — Curated list | Hardcode which metrics to show | Current PoC state — high maintenance |
| C — UCP resource-type grouping | Group by UCP concept (Database, Compute, etc.) | **Recommended long-term direction** |

**Current state** is Option B — all metrics from four monitored services are returned
without curation. Option C is the right direction for multi-cloud but requires mapping
work per provider that is out of scope for this PoC.

### Open questions for production

1. Should API rate quotas (req/min) be shown alongside resource count quotas? They serve
   different audiences (ops vs capacity planning).
2. For soft limits with no API support (Cloud SQL instance count), show usage-only with
   "limit not available", or omit the row?
3. For regional metrics, show all regions or only the tenant's configured region?
4. Should the quota page aggregate across providers (GCP + Omnia) or be per-provider?

---

## Provider-Agnostic Checklist

Use this checklist when adding quota support for a new cloud provider:

### API Availability
- [ ] Quota listing API (metric names + limits)?
- [ ] Real-time usage data via metrics/monitoring API?
- [ ] Both APIs are GA?
- [ ] Rate limits on quota APIs?

### Credentials
- [ ] Accessible with provisioning credentials already stored?
- [ ] Read-only role available?

### Resource Type Mapping

| UCP resource type | GCP quota metric | Status |
|---|---|---|
| `database` | none (soft limit, no metric ID) | **Blocked** |
| `compute` | `compute.googleapis.com/cpus` (regional) | Deferred |
| `k8s-cluster` | `container.googleapis.com/clusters` | Deferred |
| `storage` | `storage.googleapis.com/buckets` | Deferred |

### Error Handling
- [ ] HTTP status and structure when quota is exceeded during provisioning?
- [ ] Structured field to distinguish `quotaExceeded` from `rateLimitExceeded`?

---

## Success Criteria — Status

| Criterion | Status |
|---|---|
| `GET /api/v1/quota` returns real quota data using stored credentials | ✅ |
| Quota page accessible from sidebar with service filter and usage % bar | ✅ |
| `QuotaProvider` interface isolates GCP-specific logic | ✅ |
| `POST /api/v1/databases` returns HTTP 429 when Cloud SQL quota exhausted | ❌ Blocked — soft limit, no metric ID |
| Provisioning unaffected when quota is available | ✅ (gate fails open) |
