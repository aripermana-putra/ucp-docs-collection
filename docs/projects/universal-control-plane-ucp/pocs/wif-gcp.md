---
title: "WIF — GCP Workload Identity Federation with Crossplane"
space: UCP
parent_page_id: "../pocs.md"
---

# WIF — GCP Workload Identity Federation with Crossplane

| | |
|---|---|
| **Ticket** | MCUCP-217 |
| **Parent research** | [Workload Identity Federation — Feasibility for UCP](../research/workload-identity-federation-gcp.md) |
| **Status** | Complete — `Secret` + `external_account` approach works end-to-end; GP 106 requires CCoE action for production |

---

## Goal

Verify whether `provider-upjet-gcp` supports Workload Identity Federation (WIF) for a self-hosted Crossplane running on a local (non-GKE) Kubernetes cluster, and identify which credential source works end-to-end.

---

## Reading Order

1. [design.md](./wif-gcp/design.md) — scope, hypothesis, approach, and success criteria — **start here**
2. [poc-report.md](./wif-gcp/poc-report.md) — verdict after the PoC completes
3. [concepts.md](./wif-gcp/concepts.md), [implementation.md](./wif-gcp/implementation.md), [evidence.md](./wif-gcp/evidence.md) — supporting detail

---

## Sub-docs

| Document | Role | Contents |
|----------|------|----------|
| [design.md](./wif-gcp/design.md) | Human-first review | Question, hypothesis, scope, approach, success criteria, risks |
| [poc-report.md](./wif-gcp/poc-report.md) | Human-first verdict | What the PoC proved and the recommendation |
| [concepts.md](./wif-gcp/concepts.md) | Supporting context | How WIF works, credential source options, prerequisites |
| [implementation.md](./wif-gcp/implementation.md) | Supporting proof | What was built, run, and observed |
| [evidence.md](./wif-gcp/evidence.md) | Supporting evidence | Provider source code findings, external references |
