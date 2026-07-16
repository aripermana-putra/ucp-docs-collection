---
title: "Scale Baseline"
space: UCP
parent_page_id: "../production-design.md"
---

# Scale Baseline

Real-world data to ground the architecture assumptions. Data sourced from OneCloud
production Grafana (cortex-lbaas-production datasource), collected 2026-07-14.

**Status:** Incomplete — LBaaS data captured, CaaS / DBaaS / STaaS / GCP pending.

---

## Methodology

Data pulled via PromQL queries against production Grafana datasources.
All numbers are production environment, JPE region only unless noted.
JPW1 adds additional load on top of these figures.

---

## LBaaS — OneCloud Production (JPE)

### Resource count

| Metric | Value | Query |
|---|---|---|
| Total unique listeners | **28,664** | `count(count by (listener) (lbaas:haproxy_frontend_bytes_out_total:bps{environment="production"}))` |
| Total unique tenants | **1,256** | `count(count by (tenant) (lbaas:haproxy_frontend_bytes_out_total:bps{environment="production"}))` |
| Estimated LB instances | **~11,500** | listeners ÷ ~2.5 avg listeners per LB |

### Tenant distribution (top tenants by listener count)

| Tenant | Listeners | Type |
|---|---|---|
| caas | 14,539 | Service provider (K8s clusters) |
| travel | 597 | Product team |
| dbaas | 558 | Service provider |
| travel-everest | 348 | Product team |
| cpd-aps-cdn | 248 | Product team |
| gdsp-airis | 213 | Product team |
| lbaas | 208 | Service provider (own infra) |
| co-ui-api-prod | 162 | Platform team |
| mon-aas | 161 | Platform team |
| ... | ... | ... |

CaaS alone accounts for ~51% of all listeners. This reflects CaaS managing LBs
for every K8s cluster they operate across all product teams.

### Provisioning operation rate

The LBaaS control plane API (V2 + V3) peak mutation traffic — filtered to
tenant-scoped provisioning endpoints only (excludes infra/healthcheck paths
which dominated the raw traffic):

| API | Endpoint | Peak (req/s) | Notes |
|---|---|---|---|
| V3 | `PUT /v3/tenants/:tenant_name/server_groups/:name` | 1.08 | |
| V3 | `PUT /v3/tenants/:tenant_name/load_balancers/:name` | 0.30 | |
| V3 | `PUT /v3/tenants/:tenant_name/certificates/:name` | 0.017 | |
| V2 | `PUT /v2/tenants/:tenant_name/load_balancers/:name` | 0.93 | |
| V2 | `PUT /v2/tenants/:tenant_name/server_groups/:name` | 0.87 | |
| V2 | `PUT /v2/tenants/:tenant_name/gslb/:name` | 0.067 | |
| All others | DELETE, acl_rules, quotas, etc. | ~0 | Near zero in 7-day window |

**Combined peak: ~3.3 req/s → ~200 provisioning operations/minute at absolute peak.**
Average is near zero outside maintenance windows.

> Note: The raw API traffic appeared much higher (~138 req/s combined V2+V3) but
> was dominated by infra health check endpoints (`/infra/node_status`,
> `/infra/healthcheck_status_lists`) — these are internal node health reporting
> calls, not user provisioning operations.

---

## Key Findings

### The bottleneck is not provisioning throughput

Provisioning rate is low (~3 req/s peak, near zero most of the time). The
architecture document assumption of "20 provisioning requests per day" is
conservative in the right direction for Temporal workflow sizing.

### The bottleneck is managed resource scale

| Dimension | Architecture doc assumption | Reality (LBaaS JPE only) |
|---|---|---|
| Tenants | 100 | 1,256 |
| Total resources | 1,500 XR/MR objects | ~11,500 LB instances (LBaaS alone) |

This has direct implications for:
- Crossplane provider informer cache memory — holds all MR objects in memory
- Drift scan design — scanning 11,500+ MRs per GVR per minute
- etcd object count — far above the 1,500 assumed

### Service provider teams are the primary users

CaaS (14,539 listeners), DBaaS (558), LBaaS own infra (208) dominate. Product
teams are individually small but numerous. When a service provider team does a
maintenance window, they can update hundreds to thousands of resources in one batch.

---

## Pending Data

| Service | Data needed | Status |
|---|---|---|
| CaaS | Total cluster count, node count | Not yet queried |
| DBaaS | Total DB instance count | Not yet queried |
| STaaS | Total volume/bucket count | Not yet queried |
| VMaaS | Total VM count | Not yet queried |
| GCP | Resource counts per type (Cloud SQL, GKE, GCS, GCE) | Not yet queried |
| LBaaS JPW1 | Same metrics for Japan West region | Not yet queried |
