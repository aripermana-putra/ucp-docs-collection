---
title: "Scale Baseline"
space: UCP
parent_page_id: "../production-design.md"
---

# Scale Baseline

Real-world data to ground architecture assumptions and system capacity planning.
Data sourced from two approaches: Grafana PromQL queries against OneCloud production
metrics, and bottom-up analysis of architecture documents from Year 1 confirmed teams.

---

## Approach 1 — Grafana Production Metrics (LBaaS)

Data pulled via PromQL against `cortex-lbaas-production` datasource.
Production environment, JPE region only, collected 2026-07-14.

### LBaaS resource count

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
| ... | ... | ... |

CaaS alone accounts for ~51% of all listeners.

### Provisioning operation rate

LBaaS control plane API peak mutation traffic (tenant-scoped endpoints only,
health check paths excluded):

| API | Endpoint | Peak (req/s) |
|---|---|---|
| V3 | `PUT /v3/tenants/:tenant_name/server_groups/:name` | 1.08 |
| V3 | `PUT /v3/tenants/:tenant_name/load_balancers/:name` | 0.30 |
| V2 | `PUT /v2/tenants/:tenant_name/load_balancers/:name` | 0.93 |
| V2 | `PUT /v2/tenants/:tenant_name/server_groups/:name` | 0.87 |
| All others | | ~0 |

**Combined peak: ~3.3 req/s → ~200 provisioning operations/minute at absolute peak.**
Average is near zero outside maintenance windows.

---

## Approach 2 — Bottom-Up Team Analysis

### Methodology

MR count is estimated per Year 1 confirmed team using architecture documents and
Grafana data where available. Resource count depends on provisioning configuration:

- **Managed service** (CloudSQL, DBaaS) → 1 MR per instance
- **Self-managed on VMs** (Cassandra, TiDB, K8S nodes on CaaS) → 1 MR per VM/node

As a result, MR count is configuration-dependent and stated as a range rather than
a fixed number.

### Calibration sample — Budas team (small-medium product team)

Used as a baseline reference for product team sizing.

| Resource | Count | Notes |
|---|---|---|
| CaaS clusters | 3 | Shared clusters (provisioned by team) |
| DBaaS instances | 6 | |
| LBaaS (load balancers) | ~30 | |
| Event bus | 6 | Out of UCP scope (Year 1) |
| **Total UCP-relevant MRs** | **~39** | Excluding event bus |

### Architecture sample — Coupon team (large product team, Year 1 confirmed)

Architecture source: [Coupon Platform Environments and GCP Projects](https://confluence.rakuten-it.com/confluence/spaces/coupon/pages/6469558185)

Coupon runs a hybrid active-active setup: application layer split 80% GCP / 20%
OneCloud in production. Database writes go to Cloud SQL (GCP primary), with
OneCloud DBaaS serving as read replica only.

**Key resources (production environment):**

| Resource type | OneCloud | GCP | Notes |
|---|---|---|---|
| LBaaS (GSLB + DLB) | ~5 | — | |
| CaaS clusters | 2 | — | |
| Redis | 1 cluster | 1 cluster (Valkey) | Different implementations per cloud |
| DBaaS read replica | 1..N | — | OneCloud read only |
| Cloud SQL | — | 1 primary + up to 20 replicas | Writes go here |
| GKE | — | 1 autopilot cluster | |
| GCE (Valkey) | — | 1..N nodes | Self-managed |
| BMaaS batch servers | 1..N | — | |
| Kafka | 1 cluster | — | |

**Estimated UCP MR count: ~200–250 MRs** (production, OneCloud + GCP scope only)

### Architecture sample — Point team (large product team, not Year 1)

Architecture source: [One Point Platform Infrastructure Architecture](https://confluence.rakuten-it.com/confluence/spaces/ONEPOINT/pages/5255104815)

Point team runs across 4 clouds: OneCloud (JPE1, JPE2, JPW2), GCP (JPW), OCI, and Azure (AFD).
OCI and Azure are out of UCP Year 1 scope.

**Resource count (production, manual count from architecture diagram):**

| Resource | JPE1 | JPE2 | JPW2 | GCP JPW | OCI | Azure | Total | UCP scope |
|---|---|---|---|---|---|---|---|---|
| GSLB | | | | | | | 5 | ✓ |
| AFD (Azure Front Door) | | | | | | 1 | 1 | ✗ |
| Shared HWSLB | 1 | | | | | | 1 | ✗ (shared) |
| LBaaS / LB | 2 | 7 | | | | | 9 | ✓ |
| GCP LB | | | | 1 | | | 1 | ✓ |
| GCP Ingress | | | | 1 | | | 1 | ✓ |
| OCI LB | | | | | 1 | | 1 | ✗ |
| HAProxy VM/BM | 12 | 12 | | | | | 24 | ✓ |
| App Server (VM/BM) | 28 | 24 | | | 24 | | 76 | 52 in scope |
| K8S nodes | | 52 | | 1 cluster | | | 53 | ✓ |
| Cassandra nodes | | 39 | | 45 | | | 84 | ✓ |
| Apache Spark nodes | | 3 | | | | | 3 | ✓ |
| Etcd nodes | | 5 | | | | | 5 | ✓ |
| TiDB nodes | | 63 | | | | | 63 | ✓ |
| MySQL nodes | | 3 | | | | | 3 | ✓ |
| Redis nodes | | 5 | | 10 | | | 15 | ✓ |
| DB VMs | 3 | | | | | | 3 | ✓ |
| OCI ADB-D | | | | | 3 | | 3 | ✗ |
| SuperDB TeraData | 1 | | | | | | 1 | ✗ (shared) |
| Kafka nodes | | 6 | 3 | | | | 9 | ✓ |
| **Total** | | | | | | | **361** | |

**Estimated UCP MR count: ~279–330 MRs** (OneCloud + GCP scope, excluding OCI/Azure/shared infra)

The range reflects provisioning configuration dependency: self-managed VMs
(Cassandra, TiDB, K8S nodes) count per VM node; managed services count per instance.

---

## Year 1 MR Estimate — Confirmed Teams

Year 1 confirmed teams: CaaS, DBaaS, Coupon, RPay

### CaaS resource breakdown (updated from Grafana)

Data sourced from `daichi-prometheus` (BMaaS) and `cortex-caas-prod` (K8S nodes),
collected 2026-08-12.

Dashboard references:
- [CaaS Futako DB](https://monitor.rakuten-it.com/v2/d/SsdfsZofT9BHk/caas-futako-db?orgId=1&var-db=caas-futako-db-prod&var-Service_Segment_Name=All)
- [CaaS # of Nodes](https://monitor.rakuten-it.com/v2/d/jpnKe4kVz/of-nodes?orgId=1)

| Resource | Count | UCP scope | Source |
|---|---|---|---|
| LBaaS instances | ~5,800 | ✓ | Grafana: ~14,539 listeners ÷ 2.5 |
| VMaaS nodes | 7,017 | ✓ | Grafana: `count(kube_node_labels{label_machine_type="virtual"})` — CaaS # of Nodes dashboard |
| BMaaS nodes | 13,640 | ✗ Year 1 | Grafana: `count(bmaas_machine_status{status="running", tenant_name="caas"})` |
| CaaS clusters (service provider) | 160 (130 prod + 30 dev) | ✓ | CaaS Futako DB dashboard |

**CaaS as UCP tenant MR count: ~9,800–12,800 MRs**
(LBaaS ~5,800 + VMaaS 7,017, with 50% phased import = ~6,400)

> ⚠ **Disclaimer:** This estimate assumes (1) CaaS imports all existing resource
> metadata into UCP, and (2) VMaaS API (Compute API provider) is working as intended.
> VMaaS provider is currently WIP in the MVP feature list. Actual Year 1 MR count
> from CaaS depends on import scope agreed with the CaaS team and VMaaS provider
> readiness.

| Team | Type | Estimated MRs | Basis |
|---|---|---|---|
| CaaS | Service provider | ~6,400 | LBaaS ~2,900 (50%) + VMaaS ~3,500 (50%) — see disclaimer above |
| DBaaS | Service provider | ~223–4,949 | LBaaS: ~223 LBs (confirmed). Nodes: 4,726 (confirmed via `count(node_uname_info)` on cortex-dbaas-prod) but compute type unknown (VMaaS / BMaaS / RIaaS) — pending DBaaS team confirmation |
| Coupon | Large product team | ~225 | Architecture doc analysis, ~5.5× Budas sample |
| RPay | Large product team | ~350 | Larger than Coupon, payment service = higher HA requirements, ~9× Budas |
| **Year 1 Total** | | **~7,800 MRs** | |

> If CaaS imports 100% of resources and VMaaS is fully operational: **~13,400 MRs** Year 1.

---

## Year-by-Year MR Projection

| Year | Teams onboarded | Est. MR count | Notes |
|---|---|---|---|
| Year 1 | 4 (22 DevOps/SRE) | ~7,300 | CaaS dominates (~81%) |
| Year 2 | +new teams (44 DevOps/SRE) | ~10,000 | New service provider teams add more |
| Year 3 | 77 DevOps/SRE | ~20,000 | GCP resources starting |
| Year 4 | 121 DevOps/SRE | ~35,000 | App developers adding GCP resources |
| Year 5 | 154 DevOps/SRE + 170 App Dev | ~60,000 | Full scale |

---

## Key Findings

**Resource count is dominated by service provider teams, not user count.**
CaaS (~6,400–13,400 MRs depending on import scope) dominates Year 1. Point team (~330 MRs) is itself larger
than Coupon (~225 MRs). Headcount is a weak proxy for resource footprint.

**Provisioning throughput is not the bottleneck.**
LBaaS provisioning peaks at ~3.3 req/s. The binding constraint is the total MR count
driving Crossplane provider pod informer cache memory and drift scan load.

**MR count is configuration-dependent.**
Managed services (CloudSQL, DBaaS) count as 1 MR per instance. Self-managed resources
deployed on VMs (Cassandra, TiDB, K8S nodes on CaaS) count per individual VM.
MR estimates are stated as ranges, not fixed numbers.

**Multi-cloud teams are more complex than expected.**
Point team runs across 4 clouds. Large product teams have resources on clouds outside
UCP's Year 1 scope (OCI, Azure), which means some of their existing resources will
not be under UCP management initially.

**Recurring maintenance spikes are small and well-spaced.**
CaaS Slack announcement history (July 2026) shows 24 maintenance operations involving
cluster/node provisioning in one month — ~0.8 ops/day average, peak 3 ops/day at
different times. Each operation involves at most 2 clusters; the Slack announcements
mention as few as 1-2 nodes but do not state how many nodes per cluster are touched.
CaaS averages ~44 VMaaS nodes per cluster, so the actual node count per event is
unknown. The ~50 UCP API calls per event figure (covering provision submissions, status
checks, list calls, and session overhead) is an unverified estimate.

CaaS is one of four Year 1 tenants. Other tenants contribute additional maintenance
traffic estimated by scale relative to CaaS: DBaaS (~20 calls/day), Coupon (~7), RPay
(~10). Total across all tenants: ~77 calls/day average, ~200–300 on a peak day with
two tenants doing heavy maintenance simultaneously (~0.003 req/s).

Recurring maintenance is not a meaningful spike for component sizing regardless of
the exact per-event multiplier.

A potential one-time spike could occur if CaaS does a **bulk import** of existing
resources at onboarding (~12,000+ operations). However this depends entirely on
the onboarding strategy — CaaS may import incrementally, selectively, or not import
existing resources at all and only manage new ones going forward. If bulk import
happens, it can be rate-limited at import time.

Note: spike analysis covers MVP scope only (create/delete/update operations).
In-place operational tasks (stop/start/restart, patching) are out of MVP scope.
As UCP adds operational capabilities in later phases, spike frequency and volume
may increase.

**Year 1 total (~7,800 MRs) is stable across estimation approaches.**
Revising product team estimates up (Coupon 200→225, RPay 300→350) changes the total
by less than 5% because CaaS and DBaaS dominate.

---

## Capacity Sizing Rationale

UCP is sized for **Year 1 load with Year 5 architecture design.**

Year 1 load (~7,300 MRs, 22 users) is light. The architecture — sharding logic,
consistent hashing ring, multi-cluster migration strategy — is already designed to
scale to Year 5. The infrastructure provisioned at launch reflects actual Year 1
demand, not the maximum anticipated load.

**The headroom argument:**

Deploying at Year 1 sizing gives approximately 12–18 months before the first
meaningful scaling action is needed:

```
Year 1 launch:  ~7,300 MRs  ← deploy here
Year 2:        ~10,000 MRs
Year 3:        ~20,000 MRs  ← first Ops cluster scale-out likely here
```

This window is long enough to execute scaling actions properly — provisioning new
infrastructure, running migration workflows, validating — without pressure. It is
not a reason to defer planning.

**This is not "wait and see."**

The distinction:

| Wait and see (wrong) | Deploy lean + monitor (right) |
|---|---|
| Deploy, hope nothing breaks | Deploy, set alerts at 70% thresholds |
| React when limits are hit | Pre-test scaling runbook before launch |
| No lead time planning | Know provisioning lead time (e.g. CaaS dedicated cluster = days) |
| | Review actuals at 6 months |

**Scaling is triggered by observable signals**, not by calendar. When provider pod
memory crosses 70% sustained, or Temporal DB write latency degrades — those are the
triggers to execute the pre-tested scaling runbook, not to start designing one.

---

## Data Still Needed

| Team | Data needed | Status |
|---|---|---|
| CaaS | Total VM/node count for K8S cluster infrastructure | Pending |
| DBaaS | Total DB instance count | Pending |
| Coupon | Confirmed resource count from team | Outreach sent |
| Point team | Confirm K8S node vs cluster counting | Outreach pending |
| GCP | Resource counts per type across all teams (CCoE inventory) | Access request submitted (SWRSREOPE-91009) |
| LBaaS JPW1 | Same metrics for Japan West region | Not yet queried |
