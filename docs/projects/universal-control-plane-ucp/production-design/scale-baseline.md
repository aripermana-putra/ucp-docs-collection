---
title: "Scale Baseline"
space: UCP
parent_page_id: "../production-design.md"
---

# Scale Baseline

Real-world data to ground architecture assumptions and system capacity planning.
Data sourced from three approaches: Grafana PromQL queries against OneCloud production
metrics, bottom-up analysis of architecture documents from Year 1 confirmed teams, and
the CCoE GCP inventory [dashboard](https://datastudio.google.com/reporting/653f0ae9-7c10-46cf-8e4d-96e4a59901a2/page/bIroD).

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
| Cloud SQL | — | 1 primary + up to 20 replicas | Writes go here. CCoE confirmed 1 prod instance — replicas likely sub-resources, not separate MRs |
| GKE | — | 1 autopilot cluster | CCoE confirmed 1 prod cluster |
| GCE (Valkey) | — | 1..N nodes | CCoE confirmed 7 stable VMs |
| BMaaS batch servers | 1..N | — | |
| Kafka | 1 cluster | — | |

**Estimated UCP MR count: ~200–250 MRs** (production, OneCloud + GCP scope only)

GCP portion confirmed from CCoE (2026-08-17): ~9 MRs (7 GCE + 1 GKE + 1 Cloud SQL).
Remainder depends on OneCloud resources, particularly BMaaS batch server count —
pending Aniket Lambe confirmation.

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

## Approach 3 — CCoE GCP Inventory Dashboard

Data pulled from CCoE GCP inventory [dashboard](https://datastudio.google.com/reporting/653f0ae9-7c10-46cf-8e4d-96e4a59901a2/page/bIroD), collected 2026-08-17. Covers GCP
resources across all Rakuten teams. Environment breakdown is based on project name
labels — totals may not sum exactly due to projects without env labels. GCP only —
OneCloud resources (VMaaS, LBaaS, CaaS, DBaaS, STaaS) are not captured here.

### Total GCP inventory (all Rakuten teams)

| Resource type | All environments | Prod only | Notes |
|---|---|---|---|
| GCE instances | 5,941 | 2,422 | |
| GKE clusters | 500 | 146 | |
| Cloud SQL instances | 720 | 151 | |
| Cloud Functions | 1,877 | not broken down | Out of UCP scope |
| Dedicated Interconnect | 1,052 | — | See observation below |

**Cloud provider comparison (all environments, total count):**

| Resource type | GCP | Azure | AWS | GCP vs combined |
|---|---|---|---|---|
| VMs (GCE / Azure VM / EC2) | 5,941 | 2,757 | 2,557 | 5,941 vs 5,314 — comparable |
| K8s clusters (GKE / AKS / EKS) | 500 | 106 | 70 | 500 vs 176 — **~3× larger** |
| Databases (Cloud SQL / Azure DB / RDS) | 720 | 160 | 274 | 720 vs 434 — ~66% larger |
| Network (Interconnect / ExpressRoute / Direct Connect) | 1,052 | 178 | 87 | 1,052 vs 265 — **~4× larger** |

Note: Azure and AWS counts are total (no prod/non-prod breakdown available). GCP counts include all environments.

GCP is Rakuten's dominant cloud provider by resource count. Azure has the smallest
footprint of the three, particularly in K8s and network resources. The K8s and
network gap is significant — GCP's Kubernetes adoption is nearly 3× Azure and AWS
combined.

**Future scale reference:** GCP prod resources alone (2,422 + 146 + 151 = ~2,719)
represent a significant portion of the total addressable UCP market if all
Rakuten GCP-using teams are onboarded. Combined with OneCloud resources at similar
scale, Year 5 estimate of ~60,000 MRs remains plausible.

**Dedicated Interconnect observation:** Non-prod environments (QA, dev, sandbox) are
present in the data despite CCoE previously stating non-prod does not have Dedicated
Interconnect support. Not relevant to component sizing but worth noting for network
architecture planning.

### Coupon GCP (confirmed from CCoE)

| Resource type | Prod/stable count | Notes |
|---|---|---|
| GCE instances | 7 | Stable environment |
| GKE clusters | 1 | Prod environment |
| Cloud SQL instances | 1 | Prod environment |
| **Total confirmed GCP MRs** | **~9** | |

Architecture doc estimated "1 primary + up to 20 replicas" for Cloud SQL — CCoE
shows 1 instance, suggesting replicas are sub-resources rather than separate MRs.
OneCloud resources (LBaaS, CaaS, BMaaS, Redis, Kafka) remain estimated from the
architecture doc, pending Aniket Lambe confirmation.

### RPay GCP (unconfirmed ownership)

Filtered by "pay" keyword — projects likely owned by RPay but not verified:
`paydwh-computing-prod/stg`, `r-pay-1`, `rpayment-744`, `rpay-mpos-prod/stg`.

| Project | GCE | GKE | Cloud SQL |
|---|---|---|---|
| paydwh-computing-prod | 11 | 2 | 2 |
| paydwh-computing-stg | 11 | 2 | 3 |
| r-pay-1 | 150 | 9 | 5 |
| rpayment-744 | — | — | 1 |
| rpay-mpos-prod | 6 | 2 | — |
| rpay-mpos-stg | 3 | 2 | — |

**Prod-labeled only (conservative):** ~24 GCP MRs (GCE 17 + GKE 4 + Cloud SQL 3)

**Including r-pay-1 (no env label, unconfirmed):** ~188 GCP MRs (GCE 167 + GKE 13 + Cloud SQL 8)

r-pay-1 is the dominant driver (150 GCE, 9 GKE clusters) but carries no environment
label — could be all prod or a mixed environment. If confirmed as prod, GCP scope
alone (~188) would push RPay's total (GCP + OneCloud) well above the current ~350
estimate.

---

## Year 1 MR Estimate — Confirmed Teams

Year 1 confirmed teams: CaaS, DBaaS, Coupon, RPay

### DBaaS resource breakdown (confirmed from DBaaS team)

Data confirmed directly from DBaaS team, 2026-08-18.

| Resource | Production | Staging | UCP Year 1 scope | Notes |
|---|---|---|---|---|
| VMaaS nodes | 1,768 | 160 | ✓ | In scope — VMaaS provider |
| BMaaS nodes | 2,781 | 107 | ✗ Year 1 | Same as CaaS BMaaS decision |
| RIaaS nodes | 958 | 443 | ✗ Year 1 | Old VM platform; team plans migration to VMaaS ~2028. Adds to future VMaaS count when migrated. |
| Robin nodes | 6 | — | ✗ Year 1 | VM flavour, out of MVP scope |
| LBaaS instances | ~223 | — | ✓ | Confirmed via Grafana |
| **Total prod nodes** | **4,555** | **267** | | BMaaS + VMaaS + Robin |

**DBaaS as UCP tenant MR count (Year 1): ~1,991 MRs**
(LBaaS ~223 + VMaaS 1,768)

RIaaS (958 prod nodes) is a future growth data point — when migration to VMaaS completes (~2028),
DBaaS VMaaS count grows by ~958 nodes.

---

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
| DBaaS | Service provider | ~1,991 | LBaaS: ~223. VMaaS: 1,768 (confirmed, Year 1 scope). BMaaS: 2,781 (out of Year 1 scope). RIaaS: 958 prod (out of Year 1 scope, migrating to VMaaS ~2028). Data confirmed by DBaaS team 2026-08-18. |
| Coupon | Large product team | ~225 | Architecture doc + CCoE: GCP confirmed ~9 MRs (7 GCE + 1 GKE + 1 Cloud SQL). OneCloud portion (BMaaS, LBaaS, CaaS) pending Aniket Lambe confirmation — drives most of the estimate |
| RPay | Large product team | ~350–500 | Architecture doc estimate + CCoE: prod-labeled GCP ~24 MRs; including r-pay-1 (unconfirmed ownership/env) ~188 GCP MRs. If r-pay-1 is confirmed prod, total likely exceeds ~350 |
| **Year 1 Total** | | **~9,000 MRs** | DBaaS confirmed at ~1,991 (VMaaS 1,768 + LBaaS 223) |

> Conservative (CaaS 50% import, RPay at 350): **~8,600 MRs**
> Mid estimate (CaaS 50% import, RPay at 400): **~9,000 MRs**
> If CaaS imports 100% of resources and VMaaS is fully operational: **~14,800 MRs** Year 1.

---

## Year-by-Year MR Projection

| Year | Teams onboarded | Est. MR count | Notes |
|---|---|---|---|
| Year 1 | 4 (22 DevOps/SRE) | ~8,600–9,000 | Conservative ~8,600; mid ~9,000. CaaS still dominates (~72%) with DBaaS now second (~22%) |
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

**CCoE inventory data does not materially affect Year 1 sizing; DBaaS confirmation does.**
CCoE data (Coupon/RPay GCP resources) has minimal impact — product team revisions shift
the total by < 3%. However, the confirmed DBaaS node breakdown changed the Year 1 total
from ~7,800 to ~9,000 MRs (+15%), driven by 1,768 confirmed VMaaS nodes. CaaS
(~6,400 MRs) still dominates at ~71% of Year 1 total; DBaaS is now second at ~22%.

**Year 5 estimate of ~60,000 MRs is conservative, not aggressive.**
CCoE visible resource types (VM + K8s + DB) across GCP, Azure, and AWS total ~8,600
MRs. Adding resource types UCP supports but CCoE does not surface in these views
(storage, load balancers, VPC/network resources) suggests a 2–3× multiplier —
public cloud scope alone could reach ~17,000–26,000 MRs. Combined with OneCloud
(LBaaS alone has 28,664 listeners across 1,256 tenants), the Year 5 ceiling is
likely larger than the current ~60,000 estimate.

**GCP is Rakuten's dominant public cloud by resource count — prioritise GCP providers.**
GCP K8s clusters (500) are nearly 3× Azure + AWS combined (176). GCP network
interconnects (1,052) are ~4× combined (265). This validates prioritising
`provider-upjet-gcp` expansion in Year 3+ over AWS or Azure providers. Azure has the
smallest footprint of the three clouds across all resource types.

**Year 1 total is sensitive to service provider data, not product team estimates.**
Confirmed DBaaS node data (VMaaS 1,768) shifted the Year 1 total from ~7,800 to
~9,000 (+15%). By contrast, revising product team estimates (Coupon, RPay) changes
the total by < 3%. Service providers (CaaS, DBaaS) dominate — getting their data right
matters; product team precision does not.

---

## Capacity Sizing Rationale

UCP is sized for **Year 1 load with Year 5 architecture design.**

Year 1 load (~9,000 MRs, 22 users) is light. The architecture — sharding logic,
consistent hashing ring, multi-cluster migration strategy — is already designed to
scale to Year 5. The infrastructure provisioned at launch reflects actual Year 1
demand, not the maximum anticipated load.

**The headroom argument:**

Deploying at Year 1 sizing gives approximately 12–18 months before the first
meaningful scaling action is needed:

```
Year 1 launch:  ~9,000 MRs  ← deploy here
Year 2:        ~11,000 MRs
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
| DBaaS | Node counts by compute type | Confirmed 2026-08-18 — VMaaS 1,768, BMaaS 2,781, RIaaS 958, Robin 6 (prod) |
| Coupon | GCP confirmed from CCoE (~9 MRs). OneCloud resources (BMaaS, LBaaS, CaaS) pending Aniket Lambe confirmation | GCP confirmed — OneCloud pending |
| Point team | Confirm K8S node vs cluster counting | Outreach pending |
| RPay | Confirm ownership and environment of r-pay-1, paydwh, rpayment-744, rpay-mpos GCP projects | Pending |
| GCP | Resource counts per type across all teams | Confirmed — CCoE dashboard accessed 2026-08-17 |
| LBaaS JPW1 | Same metrics for Japan West region | Not yet queried |
