---
title: "Component Sizing"
space: UCP
parent_page_id: "../production-design.md"
---

# Component Sizing

Per-component resource specifications for UCP infrastructure. Sizing is based on
Year 1 load (~7,800 MRs, 22 users) with Year 5 architecture design. See
[Scale Baseline](scale-baseline.md) for the data and rationale behind these numbers.

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
- Maintenance spikes: ~50 UCP operations per CaaS event, well within capacity

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
- Trigger: CPU > 70% or p99 latency > 500ms

**Connection pools per pod:**

| Target | Connections | Notes |
|---|---|---|
| Platform DB (pgx) | 3–5 | Fast queries, low concurrency |
| Temporal Frontend | 1 | Persistent gRPC, multiplexed |
| Redis | 2–3 | Cache reads/writes |
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
Year 1: ~500 tasks (CaaS LBaaS alone: 5,800 ÷ 100 = 58 chunks)
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
~500 tasks × ~1s each ÷ 20 concurrent = ~25 seconds per scan cycle. Comfortable within 60s.

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

## Crossplane / Ops Cluster

**What drives sizing:**
- Provider pod memory — informer cache holds all MR objects of its type in memory
- Provider pod CPU — spikes during reconciliation (bulk operations, drift remediation)
- Crossplane Core — lightweight, manages compositions and functions
- Function pods — stateless gRPC servers, called during composition pipeline

**MR distribution across providers at Year 1:**

| Provider | Resource types | Estimated MRs |
|---|---|---|
| provider-roc | LBaaS, VMaaS, DBaaS, STaaS, CaaS clusters | ~6,000–13,000 (50–100% import) |
| provider-upjet-gcp | Cloud SQL, GKE, GCS, GCE | ~100–200 (Coupon GCP only, Year 1) |

**Memory calculation for provider-roc:**
```
6,000 MRs × 10KB = ~60MB informer cache (conservative 50% import)
13,000 MRs × 10KB = ~130MB informer cache (full import)
+ Go runtime + overhead = ~50–80MB
Total: 110–210MB → 512Mi limit gives comfortable headroom
```

**Pod specs:**

*Crossplane Core:*

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 250m | 500m |
| CPU limit | 500m | 1000m |
| Memory request | 256Mi | 512Mi |
| Memory limit | 512Mi | 1Gi |
| Replicas | 2 | 2 |

*provider-roc:*

| | Year 1 | Year 5 |
|---|---|---|
| CPU request | 250m | 500m |
| CPU limit | 500m | 1000m |
| Memory request | 256Mi | 1Gi |
| Memory limit | 512Mi | 2Gi |
| Replicas | 2 | 2 |

Memory limit grows Year 1→5 as MR count grows. Bump via DeploymentRuntimeConfig
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

*ESO:*

| | Year 1 |
|---|---|
| CPU request | 50m |
| CPU limit | 200m |
| Memory request | 64Mi |
| Memory limit | 128Mi |
| Replicas | 1 |

**Total Ops cluster footprint Year 1:**

| Component | Pods | Memory (requests) |
|---|---|---|
| Crossplane Core | 2 | 512Mi |
| provider-roc | 2 | 512Mi |
| provider-upjet-gcp parent | 2 | 256Mi |
| provider-gcp-sql | 2 | 256Mi |
| provider-gcp-container | 2 | 256Mi |
| provider-gcp-compute | 2 | 256Mi |
| provider-gcp-storage | 2 | 256Mi |
| Functions ×3 | 3 | 192Mi |
| ESO | 1 | 64Mi |
| **Total** | **20 pods** | **~2.5Gi** |

**Recommended Ops cluster node spec:** 3 nodes × (4 vCPU, 8GB RAM) across 3 AZs.

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

CaaS maintenance (from July 2026 Slack history): ~40 operations/day burst
Each operation = 1-2 API calls to submit

Peak: ~50-100 calls/hour during CaaS maintenance → ~0.01-0.03 req/s
Absolute ceiling: ~1-2 req/s during simultaneous maintenance batches
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

**What Platform DB stores:**

| Data | Size estimate | Notes |
|---|---|---|
| Desired state (XR spec per resource) | ~5–10KB per resource | Full spec.forProvider, all config fields |
| resource_cluster_assignments | ~200 bytes per row | Routing metadata only |
| cluster_resource_memory | ~100 bytes per row | N×T rows |
| Audit logs | ~500 bytes per row | From actual schema: UUID×2 + TEXT fields + JSONB metadata |
| Sessions | ~100 bytes per row | user_id + sid + timestamps |
| Users, RBAC, configs | negligible | Small reference tables |

Audit log row size derived from actual `audit_logs` table schema (MCUCP-129 TRD):
`id` (UUID 16B) + `user_id` (UUID 16B) + `request_id` (TEXT ~36B) + `action` (TEXT ~25B) +
`resource_type` (TEXT ~20B) + `resource_id` (TEXT ~36B) + `tenant_rns` (TEXT ~50B) +
`metadata` (JSONB 0–500B) + `created_at` (TIMESTAMPTZ 8B) + PG row overhead (~23B)
= **~230–730B, average ~500B**

**Storage calculations:**

```
Desired state:
  Year 1: 7,800 resources × 7.5KB avg = ~58MB
  Year 5: 60,000 resources × 7.5KB avg = ~450MB

Audit logs (events/day basis):
  Year 1: ~400 events/day
    - CaaS maintenance: 40 ops × 3 events = 120
    - Other provisioning: 60 ops × 3 events = 180
    - API interactions: 50 calls × 1 event = 50
    - Drift approvals: ~50
  3-year retention: 400 × 365 × 3 × 500B = ~219MB

  Year 5: ~10,000 events/day (more teams, more operations)
  3-year retention: 10,000 × 365 × 3 × 500B = ~5.4GB
  With monthly partitioning + archival after 12 months: ~1.8GB active

Other tables (sessions, RBAC, configs, assignments):
  ~50MB Year 1, ~200MB Year 5

Total storage:
  Year 1: ~330MB active → provision 20GB (generous headroom, grows slowly)
  Year 5: ~2.5GB active → provision 100GB
```

**Connections:**
```
API server:          2 pods × 5 connections = 10
provisioning-worker: 2 pods × 3 connections = 6
drift-worker:        2 pods × 3 connections = 6
Total:               ~22 connections
```
Well within PostgreSQL default max_connections (100). No pgBouncer needed.

**Write patterns:**
- Audit logs: heaviest writer, append-only sequential inserts
- Desired state writes: on every provisioning (~100/day Year 1)
- Peak writes: CaaS maintenance burst → ~150 inserts over 30 min = ~0.08 writes/s
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

*To be sized.*

---

## Redis

*To be sized.*

---

## Others (KEDA, ESO)

*To be sized.*
