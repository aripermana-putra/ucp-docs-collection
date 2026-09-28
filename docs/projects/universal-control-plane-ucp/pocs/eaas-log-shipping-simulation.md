---
title: "EaaS Log Shipping Simulation — Sandbox PoC"
space: UCP
parent_page_id: "../pocs.md"
---

# EaaS Log Shipping Simulation — Sandbox PoC

| | |
|---|---|
| **Ticket** | MCUCP-258 |
| **Parent research** | [Application Log Monitoring — Cloud Logging (GCP) vs EaaS vs Self-Hosted EFK](../research/application-log-monitoring.md) |
| **Status** | Executed — all four log groups shipped and parsed; see [poc-report.md](eaas-log-shipping-simulation/poc-report.md) |

---

## Goal

Validate that a shared Filebeat shipper (DaemonSet, not sidecar-per-pod), configured per EaaS's
documented conventions, can correctly ship **each of UCP's real log shapes** — a generic Go
service's structured JSON, Temporal Server's, Temporal Worker's, and Crossplane's own log
output — into a pipeline that mirrors EaaS's own shape (Logstash → Kafka → Elasticsearch →
Kibana), by shipping to **locally-simulated stand-ins** for that pipeline rather than the real
EaaS service.

This PoC deliberately does **not** attempt to prove GCP→EaaS network reachability. UCP's own GCP
environment is not provisioned yet, so that question cannot be tested today — it is tracked as a
direct confirmation with the EaaS team in the parent research's [Open
Questions](../research/application-log-monitoring.md#open-questions). This PoC proves the
shipper-configuration side of Option B works as expected against EaaS's documented log format and
ingestion conventions; it says nothing about whether the real network path exists or performs
well.

---

## Reading Order

1. [design.md](eaas-log-shipping-simulation/design.md) — scope, hypothesis, approach, and
   success criteria — **start here**
2. [poc-report.md](eaas-log-shipping-simulation/poc-report.md) — verdict after the PoC completes
3. [implementation.md](eaas-log-shipping-simulation/implementation.md),
   [evidence.md](eaas-log-shipping-simulation/evidence.md) — supporting detail

---

## Sub-docs

| Document | Role | Contents |
|----------|------|----------|
| [design.md](eaas-log-shipping-simulation/design.md) | Human-first review | Question, hypothesis, scope, approach, success criteria, risks |
| [poc-report.md](eaas-log-shipping-simulation/poc-report.md) | Human-first verdict | What this PoC proved for Option B's shipper side, and what it explicitly did not test |
| [implementation.md](eaas-log-shipping-simulation/implementation.md) | Supporting proof | What was built, run, and observed on the sandbox cluster and local device |
| [evidence.md](eaas-log-shipping-simulation/evidence.md) | Supporting evidence | References used during setup |
