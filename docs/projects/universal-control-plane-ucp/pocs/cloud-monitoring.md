---
title: "Cloud Monitoring (GMP) — GKE Sandbox PoC"
space: UCP
parent_page_id: "../pocs.md"
---

# Cloud Monitoring (GMP) — GKE Sandbox PoC

| | |
|---|---|
| **Ticket** | MCUCP-259 |
| **Parent research** | [System Resource Monitoring — Cloud Monitoring (GCP) vs MonaaS](../research/system-resource-monitoring.md) |
| **Status** | Executed — see [poc-report.md](cloud-monitoring/poc-report.md) for the verdict |

---

## Goal

Verify, on a real GKE cluster in the sandbox project, that Google Managed Service for
Prometheus (GMP) collects GKE resource metrics and Cloud SQL infra metrics automatically, and
scrapes a custom Prometheus-format application metric with only a `PodMonitoring` CRD — the
claims the parent research recommendation depends on.

---

## Reading Order

1. [design.md](cloud-monitoring/design.md) — scope, hypothesis, approach, and success
   criteria — **start here**
2. [poc-report.md](cloud-monitoring/poc-report.md) — verdict after the PoC completes
3. [implementation.md](cloud-monitoring/implementation.md),
   [evidence.md](cloud-monitoring/evidence.md) — supporting detail

---

## Sub-docs

| Document | Role | Contents |
|----------|------|----------|
| [design.md](cloud-monitoring/design.md) | Human-first review | Question, hypothesis, scope, approach, success criteria, risks |
| [poc-report.md](cloud-monitoring/poc-report.md) | Human-first verdict | What the PoC proved and the recommendation |
| [implementation.md](cloud-monitoring/implementation.md) | Supporting proof | What was built, run, and observed on the sandbox cluster |
| [evidence.md](cloud-monitoring/evidence.md) | Supporting evidence | GCP documentation references used during setup |
