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

## Quota Data Fetching Strategy

As UCP scales to more tenants and more cloud providers, how quota data is fetched
becomes a design decision. Four options are evaluated below.

### Option A — Proactive background sync (all tenants)

A scheduled background job fetches quota data for all registered tenants on a fixed
interval and stores it in the DB. The quota API and pre-provision gate read from the DB.

**Quantitative:**

| Metric | Value |
|---|---|
| Cloud Monitoring calls per request | 0 |
| Cloud Monitoring calls per sync run | `tenants × providers × services × 2` |
| API response latency | ~1–5 ms (DB read) |
| DB storage | ~180 KB per tenant per provider (~900 rows × 200 bytes) |
| Background infrastructure | 1 Temporal scheduled workflow |

**Qualitative pros:**
- Pre-provision gate works even during Cloud Monitoring outages
- API server request path is a cheap DB read regardless of concurrency
- Sync calls can be parallelized inside the background job
- Pattern is consistent with RBAC sync and drift detection

**Qualitative cons:**
- Syncs all registered tenants regardless of activity — wasted calls for inactive tenants
- Sync cost grows with total tenant count, not active tenant count
- Background job granularity needs a decision: one workflow for all tenants (simpler, one failure blocks others) or one per tenant (resilient, more workflow instances)

---

### Option B — TTL cache (lazy warm, serve from cache)

On first request for a tenant, fetch live from Cloud Monitoring and cache the result.
Subsequent requests within the TTL return the cached data. Cache expires and re-fetches on the next request.

**Quantitative:**

| Metric | Value |
|---|---|
| Cloud Monitoring calls on cache hit | 0 |
| Cloud Monitoring calls on cache miss | `services × 2` per tenant per provider |
| API response latency (hit) | ~1 ms |
| API response latency (miss) | ~300 ms (parallelized) |
| Infrastructure | Redis (shared) or in-memory per api-server instance |
| Memory | ~180 KB per active tenant per provider |

**Qualitative pros:**
- Only fetches data for tenants that actually use quota
- No background job

**Qualitative cons:**
- In-memory cache duplicates across api-server instances — needs Redis for consistency
- Cache miss on pre-provision gate adds ~300 ms latency and ties gate reliability to Cloud Monitoring uptime
- Cache invalidation strategy needed (TTL expiry is simple but may serve stale data at critical moments)

---

### Option C — Pure on-demand fetch

Fetch live from Cloud Monitoring every time a user requests quota data through UCP.
Calls are parallelized to reduce latency. Nothing is cached or stored.

**Quantitative:**

| Metric | Value |
|---|---|
| Cloud Monitoring calls per request | `services × 2` per provider (parallelized) |
| API response latency | ~300 ms (parallelized) |
| Infrastructure added | None |
| Storage added | None |

**Qualitative pros:**
- Always real-time data
- Simplest implementation — no cache, no background job
- Only fetches for the tenant and provider actually requested

**Qualitative cons:**
- Pre-provision gate reliability tied directly to Cloud Monitoring uptime — an outage causes the gate to fail open or return errors
- Request latency grows as more providers are added (`providers × services × 2` calls, even parallelized)
- Under high concurrent quota requests, api-server goroutine and connection count grows proportionally

---

### Option D — Lazy first fetch + background keep-warm for active tenants

On first request for a tenant, fetch live from Cloud Monitoring (parallelized), store in
DB. A background job keeps data warm only for tenants with recent activity. Inactive
tenants are not synced until they make a request again.

**Quantitative:**

| Metric | Value |
|---|---|
| Cloud Monitoring calls on warm request | 0 (DB read) |
| Cloud Monitoring calls on cold start | `services × 2` per provider (parallelized, one-time) |
| Background sync scope | Active tenants only |
| DB storage | ~180 KB per active tenant per provider |
| Infrastructure | DB + Temporal scheduled workflow (active tenants only) |

**Qualitative pros:**
- Only syncs tenants with recent activity — scales with active tenants, not total
- Pre-provision gate reliable after first warm-up (reads from DB)
- Cold start penalty is one-time per tenant, mitigated by parallelizing calls
- Combines the best of A (reliable gate, cheap reads) and C (no wasted syncs for inactive tenants)

**Qualitative cons:**
- Cold start on very first request still hits Cloud Monitoring — gate on that request fails open or blocks
- Thundering herd risk: multiple concurrent first requests for the same tenant all trigger live fetches simultaneously — requires singleflight/deduplication
- "Active tenant" definition needs an explicit rule (e.g. activity in last 7 days)
- More complex than A or C

---

### Option E — Tiered cache: Redis → DB → Cloud provider

A variation of Option B with DB as a durable backing layer. Redis serves as the hot
cache with a short TTL (minutes to hours). The DB holds the last fetched snapshot.
Cloud provider is only called when both Redis and DB data are older than a configurable
freshness threshold (e.g. 1 day for GCP — aligned with how frequently quota data actually
changes). The threshold is provider-specific since different cloud providers update quota
data at different frequencies.

**Request flow:**

```
1. Check Redis
   → hit, fresh      → return response (step 6)
   → hit, stale      → fetch from cloud provider (step 3)
   → miss            → check DB (step 2)

2. Check DB
   → fresh           → warm Redis from DB (step 5) → return response (step 6)
   → stale           → fetch from cloud provider (step 3)

3. Fetch from cloud provider (parallelized)
4. Update DB with fresh data
5. Warm Redis from DB data
6. Return response
```

A singleflight mechanism deduplicates concurrent requests for the same tenant hitting
a cold path simultaneously.

**Pre-provision gate** uses the same flow but is surgical — it only checks the specific
resource type and provider relevant to the provisioning request, not all quota data.
Freshness status is tracked at tenant + provider granularity (not per resource type —
the added precision doesn't justify the schema complexity).

**Quantitative:**

| Metric | Value |
|---|---|
| Cloud provider calls on Redis hit (fresh) | 0 |
| Cloud provider calls on DB hit (fresh) | 0 |
| Cloud provider calls on full cold start | `services × 2` per provider (parallelized) |
| API response latency (Redis hit) | <1 ms |
| API response latency (DB hit) | ~1–5 ms |
| API response latency (cold start) | ~300 ms (parallelized) |
| Infrastructure | Redis + DB |
| DB storage | ~180 KB per active tenant per provider |

**Qualitative pros:**
- Tiered degradation: Redis down → DB → cloud provider — no single point of failure
- DB serves last known data if cloud provider is unavailable at request time
- Only fetches for tenants that actually use quota (lazy)
- Pre-provision gate is fast after warm-up and surgical (single resource type + provider)
- Redis TTL and cloud provider refresh threshold are independently configurable

**Qualitative cons:**
- Most complex option — three storage layers, two independent TTLs to configure
- Requires singleflight for concurrent cold-start requests on the same tenant
- Cloud provider refresh threshold must be defined and maintained per provider as new providers are added
- Slightly more infrastructure than Option D (Redis in addition to DB + Temporal)

---

### Comparison

| Dimension | A | B | C | D | E |
|---|---|---|---|---|---|
| Cloud provider calls per request | 0 | 0 (hit) / ~8 (miss) | ~8 (always) | 0 (warm) / ~8 (cold) | 0 (Redis/DB hit) / ~8 (cold) |
| Syncs inactive tenants | Yes | No | No | No | No |
| Pre-provision gate reliability | ✅ Always | ❌ Cache-dependent | ❌ Provider-dependent | ✅ After warm-up | ✅ After warm-up |
| Resilience to cloud provider outage | ✅ | ❌ On miss | ❌ | ✅ (DB fallback) | ✅ (DB fallback) |
| Cold start penalty | None | First request per TTL | Every request | First request ever | First request ever |
| Infrastructure added | DB + Temporal job | Redis | None | DB + Temporal job | Redis + DB |
| Complexity | Medium | Medium | Low | Medium–High | High |
| Scales with provider count | ✅ | ❌ | ❌ | ✅ | ✅ |

**Recommendation: Option C for early MVP, Option E as the target architecture.**

Option C is the simplest and is what the PoC implements — sufficient for early MVP
with a small number of active tenants.

As tenant count and provider count grow, Option E is the target:
- Redis hit path is sub-millisecond for active tenants
- DB fallback means the pre-provision gate stays reliable during cloud provider outages
- Only fetches for active tenants (unlike A)
- Lazy cold start means inactive tenants never generate unnecessary cloud provider calls
- The independently configurable Redis TTL and provider refresh threshold allow fine-tuning
  per provider as UCP adds AWS, Azure, and others with different quota data update frequencies

Option D is a valid intermediate step if Redis adoption is not yet ready — it achieves
Option E's DB resilience and active-tenant-only sync without the Redis layer.

---

## Open Questions

1. **UCP platform soft quota design** — the full design of the platform soft quota
   layer is unresolved: schema, enforcement mechanism, who sets limits and how, quota
   increase workflow, display, cross-provider model, and equivalent data sources for
   providers other than GCP. These will be worked out during MVP implementation.
