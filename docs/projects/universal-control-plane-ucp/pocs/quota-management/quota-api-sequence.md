---
title: "Quota API — Detailed Call Sequence"
space: UCP
parent_page_id: "../quota-management.md"
---

# Quota API — Detailed Call Sequence

`GET /api/v1/quota?tenantId=<rns>`

---

## Sequence Diagram

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
        Handler->>Keycloak: POST /protocol/openid-connect/token<br/>grant_type=refresh_token, refresh_token=&lt;decrypted&gt;
        Keycloak-->>Handler: new access_token + refresh_token
        Handler->>SessionDB: UpdateSessionTokens(session_id, new_enc_tokens, extended_expiry)
    end

    Note over Handler: Principal{email, access_token} injected into request context

    Note over Handler: ListQuota handler starts

    Handler->>Handler: tenantID = query["tenantId"]<br/>→ "rns:rakuten:ucp:xxx"

    Note over Handler: requireTenantAdmin() check

    Handler->>Handler: extractBearerToken(r) → reads Authorization header<br/>extractEmailFromJWT(token) → reads "email" claim from JWT payload

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

    Handler->>GCPAuth: POST /token<br/>grant_type=urn:ietf:params:oauth2:grant-type:jwt-bearer<br/>assertion=&lt;signed SA JWT&gt;<br/>scope=https://www.googleapis.com/auth/cloud-platform
    Note over Handler,GCPAuth: TLS InsecureSkipVerify=true (corporate proxy)
    GCPAuth-->>Handler: {access_token: "ya29.xxx", expires_in: 3600}

    loop for each service in [sqladmin, compute, container, storage] — 8 total Monitoring calls

        Handler->>Monitoring: GET /v3/projects/my-gcp-project-qa/timeSeries
        Note over Handler,Monitoring: filter=metric.type="serviceruntime.googleapis.com/quota/limit"<br/>  AND resource.labels.service="&lt;service&gt;"<br/>interval.startTime=&lt;now-25h&gt;<br/>interval.endTime=&lt;now&gt;<br/>aggregation.alignmentPeriod=86400s<br/>aggregation.perSeriesAligner=ALIGN_NEXT_OLDER<br/>pageSize=1000<br/>Authorization: Bearer ya29.xxx

        Monitoring-->>Handler: timeSeries[]{metric.labels{quota_metric, limit_name},<br/>resource.labels{location}, points[0].value.int64Value}
        Note over Handler: Keep minimum value per (quota_metric, location)<br/>to handle multiple limit_name rows for same key

        Handler->>Monitoring: GET /v3/projects/my-gcp-project-qa/timeSeries
        Note over Handler,Monitoring: filter=metric.type="serviceruntime.googleapis.com/quota/allocation/usage"<br/>  AND resource.labels.service="&lt;service&gt;"<br/>(same interval + aggregation params)

        Monitoring-->>Handler: timeSeries[]{metric.labels{quota_metric},<br/>resource.labels{location}, points[0].value.int64Value}
        Note over Handler: Join on (quota_metric, location):<br/>limitAvailable = limit > 0 AND limit < 10^15<br/>percentage = usage/limit*100 if limitAvailable

    end

    Handler-->>Browser: 200 OK
    Note over Handler,Browser: {"count": 842, "quotas": [<br/>  {"service": "Compute Engine",<br/>   "metricName": "compute.googleapis.com/cpus",<br/>   "dimension": "asia-east1",<br/>   "limit": 5000, "usage": 3,<br/>   "limitAvailable": true, "percentage": 0.06},<br/>  ...<br/>]}

    Browser->>Browser: setQuotas(data.quotas)<br/>render table: service / metric / region / limit / usage / usage%
```

---

## Key Parameters

### Browser → API Server

| Parameter | Location | Value | Notes |
|-----------|----------|-------|-------|
| `tenantId` | Query string | `rns:rakuten:ucp:xxx` | Tenant RNS identifier |
| `X-Environment` | Header | `QA` or `PRODUCTION` | Drives secret name suffix and Horizon URL |
| `ucp-session` | Cookie | session UUID | Server-side BFF session |
| `Authorization` | Header | `Bearer <jwt>` | Used by `requireTenantAdmin` (reads raw header, not context) |

### API Server PostgreSQL (BFF session store)

The api-server runs its own dedicated PostgreSQL deployment (`postgres` in `temporal-system` namespace).
This is **not** Temporal's PostgreSQL. It stores:

| Table | Purpose |
|-------|---------|
| `sessions` | Encrypted access + refresh tokens per browser session |
| `users` | User accounts synced from Keycloak on first login |
| `identity_providers` | Keycloak realm configs per environment |
| `audit_logs` | Action audit trail (database create/delete events) |
| `blueprint_templates` | Saved HCL blueprint templates |

### X-Environment routing

| Header value | `env` after normalisation | Horizon API URL | Credential secret suffix |
|---|---|---|---|
| `QA` | `qa` | `https://qa-horizon-data-api.r-local.net/v0` | `-qa` |
| `PRODUCTION` (or anything else) | `production` | `https://horizon-data-api.r-local.net/v0` | `-production` |

> **Note:** `isUserTenantAdmin` reads `r.Header.Get("X-Environment")` directly and passes the raw value to `getHorizonAPIBaseURL()`. It does NOT call `getEnvironment()`. The switch in `getHorizonAPIBaseURL` is `strings.ToUpper(environment) == "QA"`, so `"QA"` and `"qa"` both hit the QA Horizon endpoint.

### Credential secret name

```
credentialSecretName("gcp", tenantID, env)
→ fmt.Sprintf("cloud-credentials-gcp-%s-%s", sanitizeTenantID(tenantID), env)

Namespace: crossplane-system
Fields:    credentials.json  (base64 GCP service account JSON)
           project_id        (base64 GCP project ID)
```

### Cloud Monitoring — exact call

```
GET https://monitoring.googleapis.com/v3/projects/{projectID}/timeSeries
  ?filter=metric.type="{metricType}" AND resource.labels.service="{service}"
  &interval.startTime={now-25h RFC3339}
  &interval.endTime={now RFC3339}
  &aggregation.alignmentPeriod=86400s
  &aggregation.perSeriesAligner=ALIGN_NEXT_OLDER
  &pageSize=1000
```

Called **2 × 4 = 8 times** per quota request. Paginated (`nextPageToken`) but a full page is typically returned in one call.

### Join and sentinel logic

```
key             = (metric.labels.quota_metric, resource.labels.location)
limits[key]     = min(all int64Values for that key)   ← quota/limit has limit_name duplicates
usage[key]      = single int64Value                   ← quota/allocation/usage has no duplicates

limitAvailable  = limit > 0 AND limit < 1_000_000_000_000_000   ← filters INT64_MAX sentinel
percentage      = float64(usage) / float64(limit) * 100          ← only when limitAvailable
```

---

## Known Issue

`requireTenantAdmin` calls `extractBearerToken(r)` which reads the raw `Authorization: Bearer` header. The SessionMiddleware puts the decrypted access token in `r.Context()` (as `Principal.AccessToken`), but `requireTenantAdmin` does **not** read from context. This means:

- If the browser sends only a session cookie (no `Authorization` header), the admin check returns `"no authorization token provided"` → HTTP 403.
- The frontend's `useAuthFetch` hook sets `X-Environment` but does not inject `Authorization` for cookie-authenticated sessions.
- This would silently fail for any user authenticated purely via the session cookie flow.
