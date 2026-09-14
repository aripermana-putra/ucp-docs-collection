---
title: "Temporal Multi-Cluster Replication"
space: UCP
parent_page_id: "../pocs.md"
---

# Temporal Multi-Cluster Replication

| | |
|---|---|
| **Related ticket** | [MCUCP-307](https://jira.rakuten-it.com/jira/browse/MCUCP-307) — Feasibility study for DR scenario with temporal server |
| **Status** | Complete — see [poc-report.md](temporal-multi-cluster-replication/poc-report.md) |

## Goal

Verify whether self-hosted Temporal's Multi-Cluster Replication (MCR) works end to end
locally — cluster-to-cluster connectivity, namespace and history replication, a manual
failover, and recovery from a real outage — to validate the direction recommended in the
[parent research doc](../research/temporal-multi-cluster-replication.md).

## Reading Order

1. [design.md](temporal-multi-cluster-replication/design.md) — scope, hypothesis, approach,
   and success criteria — **start here**
2. [poc-report.md](temporal-multi-cluster-replication/poc-report.md) — verdict and
   recommendation
3. [implementation.md](temporal-multi-cluster-replication/implementation.md) — full proof
   detail

## Sub-docs

| Document | Role | Contents |
|----------|------|----------|
| [design.md](temporal-multi-cluster-replication/design.md) | Human-first review | Question, hypothesis, scope, approach, success criteria, risks |
| [poc-report.md](temporal-multi-cluster-replication/poc-report.md) | Human-first verdict | What was proven, what wasn't, recommendation |
| [implementation.md](temporal-multi-cluster-replication/implementation.md) | Supporting proof | What is built, run, and verified |
