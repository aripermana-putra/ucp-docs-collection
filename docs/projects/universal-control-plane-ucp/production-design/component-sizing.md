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

Worker concurrency config (both workers):
- `MaxConcurrentWorkflowTaskExecutionSize: 10`
- `MaxConcurrentActivityTaskExecutionSize: 10`

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

## Platform DB

*To be sized.*

---

## Temporal DB

*To be sized.*

---

## Redis

*To be sized.*

---

## Others (KEDA, ESO)

*To be sized.*
