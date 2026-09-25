---
title: "Multi-Cluster ProviderConfig Sync"
space: UCP
parent_page_id: "../pocs.md"
---

# Multi-Cluster ProviderConfig Sync

| | |
|---|---|
| **Ticket** | [MCUCP-306](https://jira.rakuten-it.com/jira/browse/MCUCP-306) |
| **Parent research** | [Multi-Cluster ProviderConfig Sync via GCP Secret Manager](../research/multi-cluster-provider-config-sync.md) |
| **Status** | Complete — GitOps pull via ArgoCD `ApplicationSet` matrix generator confirmed; see [poc-report.md](multi-cluster-provider-config-sync/poc-report.md) |

---

## Goal

Validate the GitOps pull mechanism — GCP Secret Manager as the credential source of truth, ESO as the in-cluster sync bridge, ArgoCD `ApplicationSet` matrix generator as the cluster fan-out mechanism — for keeping `ExternalSecret` + `ProviderConfig` pairs correct across a growing set of Crossplane clusters.

---

## Reading Order

1. [design.md](multi-cluster-provider-config-sync/design.md) — question, hypothesis, scope, approach, success criteria — **start here**
2. [poc-report.md](multi-cluster-provider-config-sync/poc-report.md) — verdict after the PoC completed
3. [implementation.md](multi-cluster-provider-config-sync/implementation.md) — environment, GitOps structure, verification results

---

## Sub-docs

| Document | Role | Contents |
|---|---|---|
| [design.md](multi-cluster-provider-config-sync/design.md) | Human-first review | Question, hypothesis, scope, approach, success criteria, risks |
| [poc-report.md](multi-cluster-provider-config-sync/poc-report.md) | Human-first verdict | What the PoC proved, what it did not prove, recommendation |
| [implementation.md](multi-cluster-provider-config-sync/implementation.md) | Supporting proof | Environment, cluster registration, GitOps structure, verification results |
