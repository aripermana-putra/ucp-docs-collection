---
title: "MonaaS OTel Collector Variant — Sandbox PoC"
space: UCP
parent_page_id: "../pocs.md"
---

# MonaaS OTel Collector Variant — Sandbox PoC

| | |
|---|---|
| **Ticket** | MCUCP-259 |
| **Parent research** | [System Resource Monitoring — Cloud Monitoring (GCP) vs MonaaS](../research/system-resource-monitoring.md) |
| **Related PoC** | [Cloud Monitoring (GMP) — GKE Sandbox PoC](cloud-monitoring.md) — the Option A counterpart; this PoC's numbers are compared against that PoC's numbers in the parent research, not within this PoC itself |
| **Status** | Design — not yet executed |

---

## Goal

Measure, on the sandbox GKE cluster, the operational overhead of Option B's self-managed
OTel Collector variant (component count, manifest count, wall-clock setup time) for both
GKE resource metrics and a custom application metric, remote-writing to an in-cluster
Cortex-compatible backend (Grafana Mimir) standing in for MonaaS's Cortex.

This PoC covers **Option B only**. Option A's equivalent numbers come from the [Cloud
Monitoring (GMP) — GKE Sandbox PoC](cloud-monitoring.md). The side-by-side comparison of the
two, and its effect on the parent research's Recommendation, is written into the [parent
research document](../research/system-resource-monitoring.md) once both PoCs have run — not
into either PoC's own report.

---

## Reading Order

1. [design.md](monaas-otel-collector/design.md) — scope, hypothesis, approach, and success
   criteria — **start here**
2. [poc-report.md](monaas-otel-collector/poc-report.md) — verdict after the PoC completes
3. [implementation.md](monaas-otel-collector/implementation.md),
   [evidence.md](monaas-otel-collector/evidence.md) — supporting detail

---

## Sub-docs

| Document | Role | Contents |
|----------|------|----------|
| [design.md](monaas-otel-collector/design.md) | Human-first review | Question, hypothesis, scope, approach, success criteria, risks |
| [poc-report.md](monaas-otel-collector/poc-report.md) | Human-first verdict | What this PoC proved for Option B and the recommendation |
| [implementation.md](monaas-otel-collector/implementation.md) | Supporting proof | What was built, run, and observed on the sandbox cluster |
| [evidence.md](monaas-otel-collector/evidence.md) | Supporting evidence | References used during setup |
