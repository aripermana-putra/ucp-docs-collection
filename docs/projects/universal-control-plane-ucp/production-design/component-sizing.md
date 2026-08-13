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

*To be sized.*

---

## Temporal Workers

*To be sized.*

---

## Crossplane / Ops Cluster

*To be sized.*

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
