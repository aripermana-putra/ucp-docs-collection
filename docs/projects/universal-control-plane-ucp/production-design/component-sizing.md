---
title: "Component Sizing"
space: UCP
parent_page_id: "../production-design.md"
---

# Component Sizing

Per-component resource specifications for UCP infrastructure. Sizing is based on
Year 1 load (~9,000 MRs mid estimate, 22 users) with Year 5 architecture design. See
[Scale Baseline](scale-baseline.md) for the full range (~8,600–9,000) and rationale.

**Sizing philosophy:** Deploy lean (Year 1 specs), scale horizontally before
vertically. Pod spec barely changes Year 1→5 — replica count is what scales.
See [Scaling Strategy](scaling-strategy.md) for the full scaling rationale.

---

## API Server

**Workload profile:**
- 22 users Year 1 → 324 Year 5
- ~5 peak concurrent users Year 1
- QPS: negligible average, ~1–5 req/s at peak
- Stateless, async — returns 202 immediately for provisioning
- Maintenance spikes: ~50 UCP API calls per CaaS event (unverified estimate — Slack data
  gives cluster/node count but not nodes-per-cluster touched; see [Scale Baseline](scale-baseline.md));
  sizing conclusion is insensitive to the exact value

**Pod spec:**

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 100m | 250m |
| CPU limit | 500m | 500m |
| Memory request | 128Mi | 256Mi |
| Memory limit | 256Mi | 512Mi |
| Min replicas | 2 | 3 |

**HPA:**
- Min: 2 (HA, never scale below)
- Max: 8
- Trigger: CPU > 70%

**Connection pools per pod:**

| Target | Connections | Notes |
|---|---|---|
| Platform DB (pgx) | 3–5 | Fast queries, low concurrency |
| Temporal Frontend | 1 | Persistent gRPC, multiplexed |
| Redis | — | Not in MVP — deferred until second Ops cluster or ~500 tenants |
| Ops K8s API | 1 | Persistent, cross-cluster kubeconfig |

**Scale trigger:** add replicas via HPA before touching pod spec. Only bump
CPU/memory limit if p99 latency consistently degrades after HPA has already
scaled out.

---

## Temporal Server

| Service | CPU request | CPU limit | Mem request | Mem limit | Replicas (Y1) | Replicas (Y5) |
|---|---|---|---|---|---|---|
| Frontend | 250m | 500m | 256Mi | 512Mi | 2 | 3 |
| History | 500m | 1000m | 512Mi | 1Gi | 2 | 4 |
| Matching | 250m | 500m | 256Mi | 512Mi | 2 | 3 |
| Internal Worker | 250m | 500m | 256Mi | 512Mi | 2 | 2 |

History shards: 4 Year 1 (2 per instance), 8 Year 5. History is the most memory-intensive
service — caches active workflow states per shard. Scale by adding replicas (adding shards)
when History CPU > 70% sustained.

---

## Temporal Workers

**Drift scan batch design:** MR list per (GVR, tenant) is chunked into parallel
ScanTenantActivity tasks of `DRIFT_SCAN_BATCH_SIZE` (default: 100, configurable via
env var). Prevents whale tenants from blocking concurrency slots for the full scan cycle.

**Task count per scan cycle:**
```
Tasks = Σ(MRs per tenant per GVR) ÷ DRIFT_SCAN_BATCH_SIZE

Year 1 (50% CaaS import + full DBaaS, all confirmed teams):
  (LBaaS, CaaS):           ~2,900 ÷ 100 = 29 chunks
  (LBaaS, DBaaS):            ~223 ÷ 100 =  3 chunks
  (LBaaS, Coupon/RPay):       ~14 ÷ 100 =  1 chunk
  (VMaaS, CaaS):           ~3,500 ÷ 100 = 35 chunks
  (VMaaS, DBaaS):          ~1,768 ÷ 100 = 18 chunks
  (clusters, CaaS):          ~160 ÷ 100 =  2 chunks
  Others (GCE/GKE/CloudSQL for Coupon, CaaS clusters for Coupon+RPay,
          DBaaS service instances ~30, STaaS assumed <100):  ~6 chunks
  Total: ~100 chunks

Year 5: ~3,000–5,000 tasks
```

**Provisioning worker:**

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 100m | 100m |
| CPU limit | 250m | 500m |
| Memory request | 256Mi | 256Mi |
| Memory limit | 512Mi | 512Mi |
| Replicas | 2 (static) | 5 (static) |

Static replicas — always ready, no cold start. KEDA deferred until provisioning volume
justifies it (Year 3+).

**Drift worker:**

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 250m | 250m |
| CPU limit | 500m | 1000m |
| Memory request | 256Mi | 512Mi |
| Memory limit | 512Mi | 1Gi |
| Replicas | 2 (static) | HPA/KEDA 2→20 |

Year 1: 2 static replicas × 10 concurrent = 20 concurrent tasks.
~100 chunks × ~1s each ÷ 20 concurrent = ~5 seconds per scan cycle. Comfortable within 60s.

KEDA introduced when task count per scan grows beyond what static replicas can complete
within 60 seconds — expected around Year 3–4 as tenant count and MR count grow.

**Worker concurrency config (both workers):**
```go
workerOptions := worker.Options{
    MaxConcurrentActivityTaskExecutionSize: 10,
    MaxConcurrentWorkflowTaskExecutionSize: 10,
    MaxConcurrentActivityTaskPollers:       2,
    MaxConcurrentWorkflowTaskPollers:       2,
}
```

`MaxConcurrentActivityTaskExecutionSize` controls how many activity goroutines run
simultaneously per pod. Since ScanTenantActivity is I/O-bound (K8s GET calls),
goroutines spend most time waiting — higher concurrency does not significantly
increase CPU.

**Concurrency is the first scaling lever before KEDA:**

| Concurrency per pod | Year 5 scan time (600 chunks, 2 pods, 2s/chunk) |
|---|---|
| 10 (default) | 600 ÷ 20 × 2s = **60s** ← borderline |
| 20 | 600 ÷ 40 × 2s = **30s** ✓ |
| 50 | 600 ÷ 100 × 2s = **12s** ✓ |

Increase `MaxConcurrentActivityTaskExecutionSize` before reaching for KEDA. The
trade-off: more concurrent activities = more simultaneous K8s API calls = more load
on the Ops cluster kube-apiserver. Start at 10, tune up based on observed
kube-apiserver performance.

**KEDA trigger point:** when queue depth does not return to zero between scan cycles
(i.e. tasks from scan N are still pending when scan N+1 fires). Expected around
Year 4-5 depending on MR growth and concurrency tuning.

---

## Crossplane

**What drives sizing:**
- Provider pod memory — informer cache holds all MR objects of its type in memory
- Provider pod CPU — spikes during reconciliation (bulk operations, drift remediation)
- Crossplane Core — lightweight, manages compositions and functions
- Function pods — stateless gRPC servers, called during composition pipeline

**Leader election:** All provider pods (Crossplane Core and each sub-provider) run with
2 replicas and use leader election — only 1 pod actively reconciles at a time. The
standby holds the lease and is ready to take over immediately on leader failure.
Both pods maintain the full informer cache, so memory sizing applies to both replicas.
CPU sizing reflects the active leader's needs; the standby uses negligible CPU.

**MR distribution across providers at Year 1:**

| Provider | Resource type | Estimated MRs (Year 1) |
|---|---|---|
| provider-roc-lbaas | LBaaS | ~2,900–5,800 (50–100% CaaS import) + ~223 DBaaS |
| provider-roc-vmaas | VMaaS | ~3,500–7,017 CaaS (50–100% import) + 1,768 DBaaS = ~5,268–8,785 total |
| provider-roc-dbaas | DBaaS service instances | TBD — represents DBaaS product instances consumed by tenant teams (e.g. MySQL, PostgreSQL instances provisioned via DBaaS API). Distinct from DBaaS's own infrastructure (VMaaS nodes → provider-roc-vmaas, LBaaS → provider-roc-lbaas). Count unknown at Year 1. |
| provider-roc-staas | STaaS | unknown — assumed small |
| provider-roc-caas | CaaS clusters | ~160 (130 prod + 30 dev) |
| provider-upjet-gcp | Cloud SQL, GKE, GCS, GCE | ~100–200 (Coupon GCP only, Year 1) |

**Memory calculation per ROC sub-provider:**

provider-roc follows the same sub-provider design as provider-upjet-gcp — one pod per
resource type, no parent pod. Each sub-provider holds an informer cache only for its
own resource type.

```
provider-roc-lbaas:  ~6,000 MRs × 10KB + ~30MB overhead = ~90MB  → 256Mi limit
provider-roc-vmaas:  ~8,785 MRs × 10KB + ~30MB overhead = ~118MB → 256Mi limit  (CaaS 7,017 + DBaaS 1,768)
provider-roc-dbaas:  count TBD (DBaaS service instances, not infrastructure nodes) → 256Mi limit as placeholder
provider-roc-staas:  small (unknown count) + ~30MB        = ~40MB  → 128Mi limit
provider-roc-caas:   ~160 clusters × 10KB + ~30MB        = ~32MB  → 128Mi limit
```

Go runtime overhead applies per pod — total memory is higher than a single combined
pod, but each sub-provider is independently scalable and isolated. A LBaaS
reconciliation spike does not affect VMaaS.

**Pod specs:**

*Crossplane Core:*

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 250m | 500m |
| CPU limit | 500m | 1000m |
| Memory request | 256Mi | 512Mi |
| Memory limit | 512Mi | 1Gi |
| Replicas | 2 | 2 |

*ROC sub-providers — heavy (provider-roc-lbaas, provider-roc-vmaas, provider-roc-dbaas):*

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 100m | 250m |
| CPU limit | 250m | 500m |
| Memory request | 128Mi | 512Mi |
| Memory limit | 256Mi | 1Gi |
| Replicas | 2 | 2 |

*ROC sub-providers — light (provider-roc-staas, provider-roc-caas):*

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 50m | 100m |
| CPU limit | 150m | 250m |
| Memory request | 64Mi | 128Mi |
| Memory limit | 128Mi | 256Mi |
| Replicas | 2 | 2 |

Memory limits grow Year 1→5 as MR count grows. Bump via DeploymentRuntimeConfig
before considering a second Ops cluster.

*GCP sub-providers (per sub-provider):*

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 100m | 250m |
| CPU limit | 250m | 500m |
| Memory request | 128Mi | 256Mi |
| Memory limit | 256Mi | 512Mi |
| Replicas | 2 | 2 |

*Composition Functions (per function — go-templating, auto-ready, env-configs):*

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 100m | 100m |
| CPU limit | 250m | 500m |
| Memory request | 64Mi | 128Mi |
| Memory limit | 128Mi | 256Mi |
| Replicas | 1 | 2 |

**Total Crossplane footprint Year 1:**

| Component | Pods | Memory (requests) |
|---|---|---|
| Crossplane Core | 2 | 512Mi |
| provider-roc-lbaas | 2 | 256Mi |
| provider-roc-vmaas | 2 | 256Mi |
| provider-roc-dbaas | 2 | 256Mi |
| provider-roc-staas | 2 | 128Mi |
| provider-roc-caas | 2 | 128Mi |
| provider-upjet-gcp parent | 2 | 256Mi |
| provider-gcp-sql | 2 | 256Mi |
| provider-gcp-container | 2 | 256Mi |
| provider-gcp-compute | 2 | 256Mi |
| provider-gcp-storage | 2 | 256Mi |
| Functions ×3 | 3 | 192Mi |
| **Total** | **27 pods** | **~3.2Gi** |

ESO runs on the same cluster but is a separate operator — see [ESO](#eso).

**Recommended node spec for the Ops cluster:** 3 nodes × (4 vCPU, 8GB RAM) across 3 AZs.

**Scale trigger:** provider-roc memory > 70% sustained → increase memory limit via
DeploymentRuntimeConfig. Only add a second Ops cluster when vertical scaling is
exhausted. See [Scaling Strategy](scaling-strategy.md) for the full horizontal
scaling plan.

---

## API Request Rate

**Basis:** MVP has limited interactive features. Drift detection runs automatically,
provisioning is infrequent. A DevOps/SRE engineer's typical BAU day with UCP at MVP:
- Submit provisioning request: ~once a week per person
- Check workflow status: 2-3 times when something is in-flight
- Approve drift alert: rare, only when drift is detected
- List resources: occasional

```
Active users on any given day: 5-10 of 22
Average API calls per active user: ~3-5
Total: ~25-50 calls/day

Maintenance traffic per tenant (CaaS from Slack history; others estimated by scale):
  CaaS  (service provider, large):  0.8 events/day × ~50 calls = ~40 calls/day avg
  DBaaS (service provider, smaller): ~50% of CaaS cadence         = ~20 calls/day avg
  Coupon (product team, ~225 MRs):  ~4 events/month               =  ~7 calls/day avg
  RPay  (product team, ~350 MRs):   ~6 events/month               = ~10 calls/day avg
  Total across all tenants:                                        = ~77 calls/day avg

  ~50 calls/event is an unverified estimate (see scale-baseline.md)
  Peak (2 tenants with heavy maintenance same day): ~200–300 calls/day → ~0.003 req/s
```

API server is sized for **HA, not traffic.** 2 static pods is justified by fault
tolerance, not by load.

**Temporal workers also access Platform DB directly:**

| Worker | Platform DB operations |
|---|---|
| provisioning-worker | Read desired state (XR spec), write status, read NotificationConfig, write audit logs |
| drift-worker | Read resource_cluster_assignments, read NotificationConfig, write audit logs |

Both workers need Platform DB connections — included in connection pool totals.

---

## Platform DB

**What Platform DB stores (MVP implementation — PRD, TRD, and future plan):**

| Table | Size estimate | Notes |
|---|---|---|
| `identity_providers` | negligible | OIDC provider config, few rows |
| `tenants` | negligible | 4 rows Year 1, grows slowly |
| `users` | ~300 bytes/row | 22 rows Year 1, 324 Year 5 |
| `sessions` | ~300 bytes/row | ~44 active rows Year 1 (22 users × ~2 sessions) |
| `audit_logs` | ~400–500 bytes/row | Main growth table — see below |
| `blueprint_templates` | ~5–50KB/row (JSONB) | ~20–50 templates at MVP = ~1MB total, negligible |
| `api_exposures` | negligible | Per-tenant API config |
| `resource_cluster_assignments` | ~200 bytes/row | Resource routing (from scaling strategy design) |
| `cluster_resource_memory` | ~100 bytes/row | N×T rows, for sharding |
| Desired state (XR spec per resource) | ~5–10KB/row | Full spec.forProvider stored as JSONB |
| Quota management tables | TBD | Schema not finalized — per-tenant quota definitions and usage tracking |
| Policy tables | TBD | Schema not finalized — policy rules and assignments per tenant/resource |

**Note:** No RBAC table at MVP. Quota and policy storage estimates added once schema is finalized.

Audit log row size derived from actual `audit_logs` table (MVP `db.go` migrations):
`id` (UUID 16B) + `user_id` (UUID 16B) + `session_id` (TEXT ~36B) + `action` (TEXT ~25B) +
`resource` (TEXT ~30B) + `metadata` (JSONB 0–500B) + `ip_address` (TEXT ~20B) +
`user_agent` (TEXT ~80B) + `created_at` (TIMESTAMPTZ 8B) + PG row overhead (~23B)
= **~250–800B, average ~500B**

**Storage calculations:**

```
Desired state:
  Year 1: 9,000 resources × 7.5KB avg = ~68MB
  Year 5: 60,000 resources × 7.5KB avg = ~450MB

Audit logs (events/day basis):
  Year 1: ~400 events/day
    - CaaS maintenance: ~40 ops/day avg × 3 events = 120
    - DBaaS/Coupon/RPay maintenance: ~37 ops/day combined × 3 events = 111
    - API interactions: 50 calls × 1 event = 50
    - Drift approvals: ~50
  (rounds to ~330; using 400 as a conservative ceiling)
  3-year retention: 400 × 365 × 3 × 500B = ~219MB

  Year 5: ~10,000 events/day (more teams, more operations)
  3-year retention: 10,000 × 365 × 3 × 500B = ~5.4GB
  With monthly partitioning + archival after 12 months: ~1.8GB active

Other tables (sessions, users, tenants, identity_providers, api_exposures, resource_cluster_assignments, cluster_resource_memory):
  ~50MB Year 1, ~200MB Year 5

Total storage:
  Year 1: ~330MB active → provision 20GB (generous headroom, grows slowly)
  Year 5: ~2.5GB active → provision 100GB
```

**Connections:**
```
                     Year 1                      Year 5 (peak)
API server:          2 pods × 5 = 10             3 pods × 5 = 15
provisioning-worker: 2 pods × 3 =  6             5 pods × 3 = 15
drift-worker:        2 pods × 3 =  6             20 pods × 3 = 60  (KEDA peak)
Total:               ~22                          ~90
```

Year 1 is well within PostgreSQL default `max_connections` (100). At Year 5 peak
(KEDA-scaled drift-worker), raise `max_connections` to 200 — connection overhead is
~5–10MB RAM per connection, well within the 8GB Year 5 spec. pgBouncer is not needed:
all clients use pgx persistent connection pools (long-lived), so pgBouncer's benefit
of amortizing short-lived connection cost does not apply. Revisit only if connection
count grows beyond ~300.

**Write patterns:**
- Audit logs: heaviest writer, append-only sequential inserts
- Desired state writes: on every provisioning (~100/day Year 1)
- Peak writes: all-tenant maintenance peak → ~330 audit events concentrated in 30 min = ~0.18 writes/s
- Average writes: ~0.003 writes/s — extremely low IOPS requirement

**Pod / instance spec:**

| | Year 1 | Year 5 |
|---|---|---|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 4GB | 8GB |
| Storage | 20GB | 100GB |
| Replicas | Primary + Sync Standby | Primary + Sync Standby + Async Replica |

RAM basis: PostgreSQL shared_buffers ~25% of RAM. Year 1 active data (~330MB)
fits entirely in 1GB shared_buffers — 4GB RAM provides comfortable OS and index
cache headroom. Year 5 with larger dataset needs 8GB.

Async replica added at Year 5 to offload read queries (resource inventory, audit
log searches) from the primary.

**Scale trigger:** storage > 70% or p99 query latency increases on audit log scans.
Monthly partition archival (already designed) keeps active table size bounded.

---

## Temporal DB

Two PostgreSQL databases on the same instance: `temporal` (default store — execution
history, task queues, timers, shard membership) and `temporal_visibility` (searchable
workflow metadata). Single namespace, 7-day retention.

**Retention rationale:** Temporal workflow history is an operational debugging tool,
not an audit trail. Platform DB audit_logs is the system of record for provisioning
events. Debugging a failed workflow happens within hours to a few days of the failure —
7 days covers weekend incidents and slow investigations without accumulating stale
history. Archival to GCS is available if long-term storage of specific workflows is
ever needed.

**Storage drivers:**

DriftScanWorkflow dominates. It runs every minute, spawns ~100 scan chunk activities
(Year 1), and each activity carries 100 MR names in its input payload (~2KB/activity).

```
Per DriftScanWorkflow execution:
  100 scan activities × ~2.2KB = ~220KB
  10 discovery activities × ~200B = ~2KB
  Workflow overhead               = ~10KB
  Total:                          ~232KB per execution

7-day retention:
  60 executions/hour × 24h × 7 days = 10,080 executions
  10,080 × 232KB = ~2.3GB

Provisioning workflows:
  ~100/day × 7 days = 700 × ~25KB = ~17MB — negligible

Drift approval workflows:
  ~70 in 7-day window × ~30KB = ~2MB — negligible

Visibility store:
  ~10,850 workflow rows × ~2KB = ~22MB — negligible

Total Year 1: ~2GB → provision 20GB
Total Year 5: drift scan grows to ~5,000 chunks → ~11MB/execution
  10,080 × 11MB = ~111GB → provision 200GB
  Enable archival to GCS before Year 3 to contain storage growth.

Note: PostgreSQL TOAST automatically compresses column values > 2KB (applies to
Temporal history blobs). Actual on-disk size will be lower, but sizing is based on
uncompressed estimates as the conservative baseline.
```

**Write rate:**
```
DriftScanWorkflow: (100+10) activities × 3 events + 2 lifecycle = 332 events/execution ÷ 60s ≈ ~6 writes/second
Provisioning:      ~100/day                                      ≈ negligible
Total:             ~25 writes/second sustained
```

Significantly more write-intensive than Platform DB (which peaks at ~0.08 writes/s).

**Connections:**

Temporal services connect to both stores. History is the most DB-intensive service.

```
Year 1 (2 replicas per service, 20 connections each — Temporal default):
  Frontend ×2:        2 × 20 = 40
  History ×2:         2 × 20 = 40
  Matching ×2:        2 × 20 = 40
  Internal Worker ×2: 2 × 20 = 40
  Total:              160 connections
```

Set `max_connections=200` on Temporal DB instance (default 100 is insufficient).

**Pod / instance spec:**

| | Year 1 | Year 5 |
|---|---|---|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 4GB | 8GB |
| Storage | 20GB SSD | 100GB SSD |
| Replicas | Primary + Sync Standby | Primary + Sync Standby |

RAM basis: shared_buffers at 1GB (Year 1) buffers active shard data and hot workflow
history. History service reads workflow state on activity resume — buffer cache hit
reduces latency on the resume path.

**Scale trigger:** storage > 70% → enable archival to GCS before adding storage.
At Year 5, drift scan chunk count grows with MR count — archival is the primary lever
before vertical storage scaling.

---

## Redis

Not included in MVP. Hash ring logic is present in code from Day 1 and handles N
clusters correctly. At launch with a single Ops cluster, the ring state is static —
API servers load it from Platform DB on startup and no cross-replica propagation
problem exists. Cluster additions are planned operational events (Year 3+), and
Platform DB polling (30s interval) is sufficient to propagate ring changes at that
cadence.

Other proposed uses (session cache, rate limiting) are handled by Platform DB and
nginx ingress at MVP scale.

Redis is introduced when either:
- Tenant count exceeds ~500 active tenants → quota cache hot tier
- A second Ops cluster is added → hash ring propagation across API server replicas

Whichever comes first. See [System Design](../system-design.md) for the full deferral rationale.

---

## Others (KEDA, ESO)

### KEDA

KEDA is deferred to Year 3–4. At Year 1 load (~500 scan chunks, 2 static drift-worker
pods, 10 concurrent activities each), the scan cycle completes in ~25 seconds — well
within the 60-second budget. Static replicas are the right choice: no cold start,
predictable behavior, simpler operations.

KEDA is introduced when the task queue depth no longer returns to zero between scan
cycles — the observable signal that static replicas can no longer keep up. See
[Temporal Workers](#temporal-workers) for the full concurrency tuning rationale and
KEDA trigger point.

**When introduced (Year 3–4), installed on the Platform cluster:**

Two components:

| Component | CPU request | CPU limit | Mem request | Mem limit | Replicas |
|---|---|---|---|---|---|
| KEDA Operator | 50m | 250m | 64Mi | 256Mi | 2 |
| KEDA Metrics API Server | 50m | 250m | 64Mi | 256Mi | 2 |

KEDA Operator watches ScaledObjects and adjusts replica counts. Metrics API Server
exposes Temporal task queue depth as a custom metric to the Kubernetes HPA.

**ScaledObject config for drift-worker:**
- Scaler: Temporal task queue depth
- Min replicas: 2
- Max replicas: 20
- Target: queue depth per pod ≤ `MaxConcurrentActivityTaskExecutionSize` (10)

---

### ESO

ESO runs on the Ops cluster as a separate operator alongside Crossplane. It watches
`ExternalSecret` resources and syncs values from GCP Secret Manager into standard
Kubernetes Secrets that Crossplane ProviderConfigs reference.

**Pod spec:**

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 50m | 50m |
| CPU limit | 200m | 200m |
| Memory request | 64Mi | 64Mi |
| Memory limit | 128Mi | 128Mi |
| Replicas | 1 | 1 |

ESO is lightweight — it only syncs secrets on a configurable refresh interval (default
1h). Memory and CPU are insensitive to MR count or tenant count at UCP's scale.

**What ESO does on the Ops cluster:**

ESO syncs secrets from GCP Secret Manager into K8s secrets that Crossplane
ProviderConfigs reference. Two distinct credential types:

| Provider | What is synced | Sensitivity |
|---|---|---|
| provider-roc-* (lbaas, vmaas, dbaas, staas, caas) | ROC/OneCloud API credentials (tokens, keys) | Sensitive — real credentials |
| provider-upjet-gcp | Tenant GCP project ID + tenant SA email | Metadata — not sensitive credentials |

For provider-upjet-gcp, UCP uses Workload Identity Federation: UCP's GCP Service
Account impersonates the tenant's GSA at runtime. No SA key is ever stored or exported
from the tenant's project. The tenant grants UCP's GSA `roles/iam.serviceAccountTokenCreator`
on their SA — access is revocable by the tenant at any time.

**How ESO authenticates to Secret Manager:**

| Ops cluster deployment | ESO auth to Secret Manager |
|---|---|
| GKE | Workload Identity — ESO pod's KSA bound to a GSA with `roles/secretmanager.secretAccessor` |
| CaaS | GCP Service Account key mounted as a K8s secret — requires key rotation management |

GKE is the preferred deployment target for the Ops cluster because Workload Identity
eliminates long-lived credentials entirely. On CaaS, the SA key requires manual
rotation and secure storage.

---

## QA Environment

QA is a separate environment with a separate OneCloud tenant per environment. It runs
integration tests, automated pipelines, and CI/CD flows — generating significantly more
provisioning churn than prod BAU.

**Sizing basis:**
- Resource count: 20–30% of prod (~1,800–2,700 MRs, single Ops cluster). DBaaS staging data (VMaaS 160 in scope) confirms staging environments are typically much leaner than 20–30% of prod (~9% for DBaaS VMaaS). The range is a planning ceiling.
- Provisioning rate: 3–4× prod (~300–400 ops/day) as a planning ceiling — QA sees more
  churn than prod from CI/CD pipelines and e2e tests (which run provision/delete cycles
  against QA, with a toggle for real vs mock cloud platform APIs). But provisioning rate
  does not drive any component spec at these volumes. Even at 10× prod (0.01 req/s), no
  component is constrained by throughput: the API server returns 202 immediately, and 1
  provisioning-worker pod with 10 concurrent activities handles thousands of provisions
  per day without saturation. The actual sizing constraint is MR count (informer cache
  for Crossplane, workflow volume for Temporal DB).
- HA not required — single replicas for most components; QA disruption is tolerable
- No sync standby for Platform DB

**Drift scan at QA scale:**
```
~1,800–2,700 MRs across ~5 GVRs = ~20–30 scan chunks per cycle
1 drift-worker pod × 10 concurrent activities = 10 concurrent tasks
~30 chunks × ~1s/chunk ÷ 10 concurrent = ~3 seconds per scan cycle — trivial
```

### API Server (QA)

| | QA |
|---|---|
| CPU request | 50m |
| CPU limit | 200m |
| Memory request | 64Mi |
| Memory limit | 128Mi |
| Replicas | 1 |

Single replica — no HA requirement. 300–400 ops/day = ~0.004 req/s average; the
async 202 design means API server is never the bottleneck even at peak test load.

### Temporal Server (QA)

| Service | CPU request | CPU limit | Mem request | Mem limit | Replicas |
|---|---|---|---|---|---|
| Frontend | 100m | 250m | 128Mi | 256Mi | 1 |
| History | 250m | 500m | 256Mi | 512Mi | 1 |
| Matching | 100m | 250m | 128Mi | 256Mi | 1 |
| Internal Worker | 100m | 250m | 128Mi | 256Mi | 1 |

History shards: 2 (minimum). Single replicas acceptable — workflow failures in QA
are tolerable and self-healing on restart.

### Temporal Workers (QA)

**Provisioning worker:**

| | QA |
|---|---|
| CPU request | 100m |
| CPU limit | 250m |
| Memory request | 128Mi |
| Memory limit | 256Mi |
| Replicas | 2 (static) |

2 replicas kept for provisioning worker — not for throughput (1 pod handles thousands
of provisions/day at this scale), but to avoid a single point of failure during active
test runs where a pod restart would stall the entire pipeline.

**Drift worker:**

| | QA |
|---|---|
| CPU request | 100m |
| CPU limit | 250m |
| Memory request | 128Mi |
| Memory limit | 256Mi |
| Replicas | 1 (static) |

1 replica sufficient — ~20 scan chunks per cycle completes in seconds at concurrency=10.

### Crossplane (QA)

All providers: 1 replica each.

| Component | CPU request | CPU limit | Mem request | Mem limit |
|---|---|---|---|---|
| Crossplane Core | 100m | 250m | 128Mi | 256Mi |
| provider-roc-lbaas | 50m | 150m | 64Mi | 128Mi |
| provider-roc-vmaas | 50m | 150m | 64Mi | 128Mi |
| provider-roc-dbaas | 50m | 150m | 64Mi | 128Mi |
| provider-roc-staas | 50m | 100m | 32Mi | 64Mi |
| provider-roc-caas | 50m | 100m | 32Mi | 64Mi |
| GCP sub-providers (each) | 25m | 50m | 32Mi | 64Mi |
| Composition Functions (each) | 50m | 100m | 32Mi | 64Mi |

Memory basis: 2,000 MRs ÷ 5 resource types = ~400 MRs per sub-provider × 10KB = ~4MB
informer cache each. 64–128Mi limits are generous.

### Platform DB (QA)

| | QA |
|---|---|
| CPU | 1 vCPU |
| RAM | 2GB |
| Storage | 10GB |
| Replicas | Single instance (no standby) |

Storage: 2,000 MRs × 7.5KB desired state = ~15MB. Audit logs at 400/day × 365 days ×
12 months × 500B = ~73MB. Total well under 1GB — 10GB provides headroom for test
data accumulation.

Audit log retention: 12 months (vs 36 months prod). QA audit data has no compliance
or operational significance.

### Temporal DB (QA)

| | QA |
|---|---|
| CPU | 1 vCPU |
| RAM | 2GB |
| Storage | 10GB |
| Replicas | Single instance |

Storage: QA drift scan executions are small — ~20 activities × ~2.2KB + overhead
≈ ~45KB/execution. 10,080 executions × 45KB = ~453MB. Well within 10GB.
7-day retention same as prod — operational debugging window is the same regardless
of environment.

### Redis (QA)

Not included — deferred alongside prod. Introduced in QA when Redis is added to prod.

### Others — KEDA, ESO (QA)

**KEDA:** Not included — deferred same as prod. Introduced in QA when KEDA is added
to prod (Year 3–4).

**ESO:**

| | QA |
|---|---|
| CPU request | 25m |
| CPU limit | 100m |
| Memory request | 32Mi |
| Memory limit | 64Mi |
| Replicas | 1 |

Same role as prod — syncs ROC credentials and GCP tenant metadata from Secret Manager
into K8s secrets. Specs are halved from prod; QA secret sync volume is identical.

---

## Staging Environment

Staging is a pre-production environment for validating UCP changes before prod deployment.
It manages actual staging resources of tenant teams — sized closer to prod than QA.
DB standby is required to support failover testing.

**Sizing basis:**
- Resource count: ~30–40% of prod (~2,500–3,500 MRs). DBaaS staging confirmed at 160
  VMaaS in scope; CaaS staging unknown — range is an estimate pending CaaS staging data.
- HA: DB requires async standby (failover testing). K8s components run 2 replicas for
  prod-like behaviour but on fewer nodes (2 vs 3).
- Node spec matches prod (4 vCPU, 8GB) — staging must exercise the same resource
  profiles, just at smaller scale.

**Drift scan at Staging scale:**
```
~2,500–3,500 MRs across ~5 GVRs = ~30–40 scan chunks per cycle
1 drift-worker pod × 10 concurrent = ~3–4 seconds per scan cycle
```

### API Server (Staging)

| | Staging |
|---|---|
| CPU request | 100m |
| CPU limit | 500m |
| Memory request | 128Mi |
| Memory limit | 256Mi |
| Replicas | 2 |

### Temporal Server (Staging)

| Service | CPU request | CPU limit | Mem request | Mem limit | Replicas |
|---|---|---|---|---|---|
| Frontend | 250m | 500m | 256Mi | 512Mi | 1 |
| History | 500m | 1000m | 512Mi | 1Gi | 1 |
| Matching | 250m | 500m | 256Mi | 512Mi | 1 |
| Internal Worker | 250m | 500m | 256Mi | 512Mi | 1 |

History shards: 2. Single replicas — staging disruption is tolerable.

### Temporal Workers (Staging)

| | Provisioning | Drift |
|---|---|---|
| CPU request | 100m | 250m |
| CPU limit | 250m | 500m |
| Memory request | 256Mi | 256Mi |
| Memory limit | 512Mi | 512Mi |
| Replicas | 2 (static) | 1 (static) |

### Crossplane (Staging)

All providers: 1 replica each. Same node spec as prod (4 vCPU, 8GB) — exercises real
provider behaviour.

| Component | CPU request | CPU limit | Mem request | Mem limit |
|---|---|---|---|---|
| Crossplane Core | 250m | 500m | 256Mi | 512Mi |
| ROC heavy sub-providers (each) | 100m | 250m | 128Mi | 256Mi |
| ROC light sub-providers (each) | 50m | 150m | 64Mi | 128Mi |
| GCP sub-providers (each) | 100m | 250m | 128Mi | 256Mi |
| Composition Functions (each) | 100m | 250m | 64Mi | 128Mi |

### Platform DB (Staging)

| | Staging |
|---|---|
| CPU | 1 vCPU |
| RAM | 2GB |
| Storage | 10GB |
| Replicas | Primary + Async Standby |

Async standby enables failover testing without the write latency overhead of sync.

### Temporal DB (Staging)

| | Staging |
|---|---|
| CPU | 1 vCPU |
| RAM | 2GB |
| Storage | 10GB SSD |
| Replicas | Primary + Async Standby |

### Redis, KEDA, ESO (Staging)

Same deferral rules as prod. ESO same spec as QA.

---

## Dev Environment

Dev is a shared developer testing environment for validating feature changes before QA.
Minimal resources, no HA, single replicas throughout. Disruption is acceptable.

**Sizing basis:**
- Resource count: ~10–15% of prod (~500–1,000 MRs) — mostly synthetic test resources
- No HA requirements
- Smaller nodes (2 vCPU, 4GB) — dev workload is negligible

**Drift scan at Dev scale:**
```
~500–1,000 MRs across ~5 GVRs = ~5–10 scan chunks per cycle
~10 chunks × ~1s ÷ 10 concurrent = ~1 second per scan cycle
```

### API Server (Dev)

| | Dev |
|---|---|
| CPU request | 50m |
| CPU limit | 200m |
| Memory request | 64Mi |
| Memory limit | 128Mi |
| Replicas | 1 |

### Temporal Server (Dev)

| Service | CPU request | CPU limit | Mem request | Mem limit | Replicas |
|---|---|---|---|---|---|
| Frontend | 100m | 250m | 128Mi | 256Mi | 1 |
| History | 250m | 500m | 256Mi | 512Mi | 1 |
| Matching | 100m | 250m | 128Mi | 256Mi | 1 |
| Internal Worker | 100m | 250m | 128Mi | 256Mi | 1 |

History shards: 2 (minimum).

### Temporal Workers (Dev)

| | Provisioning | Drift |
|---|---|---|
| CPU request | 100m | 100m |
| CPU limit | 250m | 250m |
| Memory request | 128Mi | 128Mi |
| Memory limit | 256Mi | 256Mi |
| Replicas | 1 (static) | 1 (static) |

### Crossplane (Dev)

All providers: 1 replica each.

| Component | CPU request | CPU limit | Mem request | Mem limit |
|---|---|---|---|---|
| Crossplane Core | 100m | 250m | 128Mi | 256Mi |
| ROC sub-providers (each) | 50m | 100m | 32Mi | 64Mi |
| GCP sub-providers (each) | 25m | 50m | 32Mi | 64Mi |
| Composition Functions (each) | 50m | 100m | 32Mi | 64Mi |

### Platform DB (Dev)

| | Dev |
|---|---|
| CPU | 1 vCPU |
| RAM | 2GB |
| Storage | 5GB |
| Replicas | Single instance |

### Temporal DB (Dev)

| | Dev |
|---|---|
| CPU | 1 vCPU |
| RAM | 2GB |
| Storage | 5GB SSD |
| Replicas | Single instance |

### Redis, KEDA, ESO (Dev)

Same deferral rules as prod. ESO same spec as QA.

---

## Cluster Summary

UCP runs as a separate instance per environment. Each instance spans two K8s clusters
and two managed PostgreSQL instances.

**Environment overview:**

| | Dev | QA | Prod |
|---|---|---|---|
| Est. MR count | ~2,000–3,000 | ~9,000 | ~9,000 |
| Platform cluster | 2 nodes/zone (6 total) e2-custom-2-4096 | 3 nodes/zone (9 total) n2-custom-4-8192 | 3 nodes/zone (9 total) n2-custom-4-8192 |
| Ops cluster | 2 nodes/zone (6 total) e2-custom-2-4096 | 3 nodes/zone (9 total) n2-custom-4-8192 | 3 nodes/zone (9 total) n2-custom-4-8192 |
| API Server replicas | 1 | 2 (HPA 2–8) | 2 (HPA 2–8) |
| Temporal services | 1 each | 2 each | 2 each |
| Provisioning worker | 2 | 2 | 2 |
| Drift worker | 1 | 2 | 2 |
| Crossplane providers | 1 each | 2 each | 2 each |
| Platform DB | db-custom-1-3840, 10GB, single | db-custom-2-7680, 20GB, sync standby | db-custom-2-7680, 20GB, sync standby |
| Temporal DB | db-custom-1-3840, 10GB SSD, single | db-custom-2-7680, 20GB SSD, sync standby | db-custom-2-7680, 20GB SSD, sync standby |
| DR site | None | None | Option 2 if BCP Lvl 4 mandated |

---

### Platform Cluster (Prod)

Hosts the UCP control plane: API server, Temporal server, and Temporal workers.
Platform DB and Temporal DB run as managed instances outside this cluster.

**Year 1 workload:**

| Component | Pods | CPU requests | Mem requests |
|---|---|---|---|
| API Server | 2 | 200m | 256Mi |
| Temporal Frontend | 2 | 500m | 512Mi |
| Temporal History | 2 | 1,000m | 1,024Mi |
| Temporal Matching | 2 | 500m | 512Mi |
| Temporal Internal Worker | 2 | 500m | 512Mi |
| provisioning-worker | 2 | 200m | 512Mi |
| drift-worker | 2 | 500m | 512Mi |
| **Total** | **14 pods** | **3,400m** | **3,840Mi** |

**Recommended node spec Year 1:** 3 nodes per zone × 3 zones = 9 nodes total, n2-custom-4-8192 (4 vCPU, 8GB).

```
Allocatable per node (system reserved ~0.5 vCPU, ~1GB):
  CPU:    ~3.5 vCPU × 9 nodes = ~31.5 vCPU total
  Memory: ~13GB × 9 nodes     = ~117GB total

vs Year 1 requests: 3.4 vCPU, 3.75GB — very comfortable headroom
vs Year 1 limits:   ~8.5 vCPU, ~8.5Gi

3 nodes per zone ensures zone failure tolerance: surviving 6 nodes
comfortably absorb all pods without rescheduling pressure.
```

**Year 5 note:** KEDA drift-worker scales to 20 pods (20 × 512Mi = 10Gi requests
for drift-workers alone). The existing 9-node spec has sufficient headroom through
Year 5 — add nodes before KEDA is introduced if utilization approaches 70%.

---

### Ops Cluster (Prod)

Hosts Crossplane (all sub-providers and composition functions) and ESO.

**Year 1 workload:**

| Component | Pods | CPU requests | Mem requests |
|---|---|---|---|
| Crossplane Core | 2 | 500m | 512Mi |
| provider-roc-lbaas | 2 | 200m | 256Mi |
| provider-roc-vmaas | 2 | 200m | 256Mi |
| provider-roc-dbaas | 2 | 200m | 256Mi |
| provider-roc-staas | 2 | 100m | 128Mi |
| provider-roc-caas | 2 | 100m | 128Mi |
| provider-upjet-gcp parent | 2 | 200m | 256Mi |
| provider-gcp-sql | 2 | 200m | 256Mi |
| provider-gcp-container | 2 | 200m | 256Mi |
| provider-gcp-compute | 2 | 200m | 256Mi |
| provider-gcp-storage | 2 | 200m | 256Mi |
| Functions ×3 | 3 | 300m | 192Mi |
| ESO | 1 | 50m | 64Mi |
| **Total** | **28 pods** | **~2,650m** | **~3,372Mi** |

**Recommended node spec:** 3 nodes × (4 vCPU, 8GB RAM) across 3 AZs.

```
Allocatable: ~10.5 vCPU, ~21GB total
vs requests: 2.65 vCPU, ~3.3Gi
Headroom: 4× CPU, 6× memory — accommodates reconciliation bursts and
informer cache growth as MR count grows toward Year 5.
```

---

### Managed Instances (Prod)

| Instance | CPU | RAM | Storage | HA |
|---|---|---|---|---|
| Platform DB | 2 vCPU | 4GB | 20GB | Primary + Sync Standby |
| Temporal DB | 2 vCPU | 4GB | 20GB SSD | Primary + Sync Standby |

Both run as managed PostgreSQL (Cloud SQL or equivalent) — not K8s workloads.

**Non-prod managed instances:**

| Instance | Dev | QA |
|---|---|---|
| Platform DB | db-custom-1-3840, 10GB, single | db-custom-2-7680, 20GB, sync standby |
| Temporal DB | db-custom-1-3840, 10GB SSD, single | db-custom-2-7680, 20GB SSD, sync standby |

QA is 1:1 with prod (Option 1) — same spec, no DR site. Dev covers combined Dev + QA load with single instance DBs; disruption is tolerable.
