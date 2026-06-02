---
title: "Quota Management — Implementation"
space: UCP
parent_page_id: "../quota-management.md"
---

# Quota Management — Implementation

UCP reads real-time quota limits and usage from GCP using each tenant's stored credentials
and surfaces them through a dedicated API endpoint and UI. A pre-provision gate is wired
to block resource creation before a quota-exceeded request reaches the cloud provider.

This covers cloud provider quota visibility only. UCP platform-level soft quotas
(`quota_policies`, `CheckQuota`) are a separate concern — see `ucp-quota-design.md`.

---

## Implementation

| Component | File | Status |
|---|---|---|
| `QuotaProvider` interface | `backend/api-server/quota_handler.go` | Deployed |
| `gcpQuotaProvider` | `backend/api-server/quota_handler.go` | Deployed |
| `GET /api/v1/quota` | `backend/api-server/main.go:197` | Deployed |
| Pre-provision gate (`CheckPreProvision`) | `backend/api-server/main.go:478` | Deployed — fails open |

Both quota limits and usage are read from **Cloud Monitoring** (`serviceruntime.googleapis.com`).
No separate quota API is used. See `gcp-api-reference.md` for confirmed field structures.

---

## Scope

### In Scope

- **`GET /api/v1/quota`** — returns all quota metrics (usage + limit + percentage) for the
  tenant's GCP project across four services: Cloud SQL, Compute Engine, GKE, Cloud Storage
- **`QuotaProvider` interface** — abstracts GCP-specific logic behind a provider-agnostic
  contract; adding a new provider means implementing the interface
- **Pre-provision gate** — `CheckPreProvision` is called by `POST /api/v1/databases` before
  creating any XR or Temporal workflow; returns HTTP 429 if quota is exhausted. Fails open
  for GCP — some resource count limits have no programmatic metric ID (see Limitations)

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
    Note over Browser,Handler: Cookie: ucp-session=abc123<br/>X-Environment: QA

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

    Note over Handler: RequirePermission(PermManage) check

    Handler->>Handler: extractBearerToken(r)<br/>→ reads Principal.AccessToken from context (set by SessionMiddleware)<br/>→ falls back to Authorization header only if context is empty

    Handler->>Handler: getEnvironment(r)<br/>X-Environment: "QA" → env = "qa"

    Note over Handler,K8s: gcpHTTPClient() — resolve GCP credentials

    Handler->>Handler: credentialSecretName("gcp", tenantID, "qa")<br/>→ "cloud-credentials-gcp-[sanitized-rns]-qa"

    Handler->>K8s: GET /api/v1/namespaces/crossplane-system/secrets/<br/>cloud-credentials-gcp-[sanitized-rns]-qa
    K8s-->>Handler: Secret{data: {credentials.json: [b64], project_id: [b64]}}

    Handler->>Handler: base64-decode credentials.json → GCP service account JSON<br/>base64-decode project_id → "my-gcp-project-qa"

    Handler->>GCPAuth: POST /token (service account JWT bearer)<br/>scope=https://www.googleapis.com/auth/cloud-platform
    Note over Handler,GCPAuth: TLS InsecureSkipVerify=true for corporate proxy
    GCPAuth-->>Handler: {access_token: "ya29.xxx", expires_in: 3600}

    loop for each service in [sqladmin, compute, container, storage] — 8 total calls

        Handler->>Monitoring: GET /v3/projects/my-gcp-project-qa/timeSeries
        Note over Handler,Monitoring: filter=metric.type="serviceruntime.googleapis.com/quota/limit"<br/>  AND resource.labels.service="[service]"<br/>interval.startTime=[now-25h], interval.endTime=[now]<br/>aggregation.alignmentPeriod=86400s, perSeriesAligner=ALIGN_NEXT_OLDER<br/>pageSize=1000
        Monitoring-->>Handler: timeSeries[]{metric.labels{quota_metric, limit_name},<br/>resource.labels{location}, points[0].value.int64Value}
        Note over Handler: Keep minimum value per (quota_metric, location)<br/>to handle multiple limit_name rows for same key

        Handler->>Monitoring: GET /v3/projects/my-gcp-project-qa/timeSeries
        Note over Handler,Monitoring: filter=metric.type="serviceruntime.googleapis.com/quota/allocation/usage"<br/>  AND resource.labels.service="[service]"
        Monitoring-->>Handler: timeSeries[]{metric.labels{quota_metric},<br/>resource.labels{location}, points[0].value.int64Value}
        Note over Handler: Join on (quota_metric, location)<br/>limitAvailable = limit > 0 AND limit < 10^15<br/>percentage = usage/limit*100 if limitAvailable

    end

    Handler-->>Browser: 200 OK {"count": 842, "quotas": [...]}
    Browser->>Browser: render table: service / metric / region / limit / usage / usage%
```

### Key parameters

**X-Environment routing:**

| Header value | `env` | Horizon URL | Credential secret suffix |
|---|---|---|---|
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

## Limitations

### Some resource count limits have no programmatic metric ID

Not all GCP resource count limits are exposed as programmatic metrics. Cloud SQL
instances per project is a confirmed example — it is a soft limit with no metric ID in
any quota API and does not appear in Cloud Monitoring. The pre-provision gate therefore
fails open for this resource type — `CheckPreProvision` always returns nil.

The Cloud SQL metrics that do appear in Cloud Monitoring (`connect`, `get`, `list`,
`mutate`) are **rate quotas** on Admin API call frequency. They are not relevant to
provisioning decisions.

### Cloud Monitoring returns fewer metrics than the GCP Console

The GCP Console's Quotas & System Limits page shows more quota metrics per service than
Cloud Monitoring returns. For Cloud SQL (`sqladmin.googleapis.com`), Cloud Monitoring
returns only `connect`, `get`, `list`, and `mutate`. The Console shows these plus
additional metrics. This was observed in live calls against the sandbox project but the
exact mechanism has not been verified against GCP documentation. See
`gcp-api-reference.md` § "Coverage limitation" for detail.

### Quota data volume is large and unfiltered

A typical GCP project returns 800–900 quota rows across four services. Each metric
produces multiple rows due to regional dimensions. The current implementation returns all
rows unfiltered. The recommended long-term direction is to group by UCP resource type
(Database, Compute, Kubernetes, Storage) so the UI reflects provisioning capacity rather
than raw GCP quota topology.

### Pre-provision gate resource type mapping

| UCP resource type | GCP quota metric | Gate status |
|---|---|---|
| `database` | — (soft limit, no metric ID) | Fails open |
| `compute` | `compute.googleapis.com/cpus` (regional) | Not wired |
| `k8s-cluster` | `container.googleapis.com/clusters` | Not wired |
| `storage` | `storage.googleapis.com/buckets` | Not wired |

---

## Adding a New Cloud Provider

Implement the `QuotaProvider` interface in `quota_handler.go`:

```go
type QuotaProvider interface {
    ListQuotas(ctx context.Context, tenantID, env string) ([]QuotaEntry, error)
    CheckPreProvision(ctx context.Context, tenantID, env, resourceType string) error
}
```

Before implementing, answer this checklist:

**API availability**
- [ ] Quota listing API that returns metric names and limits?
- [ ] Real-time usage data via a monitoring/metrics API?
- [ ] Both APIs are GA?

**Credentials**
- [ ] Accessible with the provisioning credentials already stored for the provider?

**Resource type mapping** — what is the exact metric ID for each UCP resource type?
- [ ] `database`
- [ ] `compute`
- [ ] `k8s-cluster`
- [ ] `storage`

**Error handling**
- [ ] What HTTP status does the provider return when quota is exceeded during provisioning?
- [ ] Structured field to distinguish `quotaExceeded` from `rateLimitExceeded`?
