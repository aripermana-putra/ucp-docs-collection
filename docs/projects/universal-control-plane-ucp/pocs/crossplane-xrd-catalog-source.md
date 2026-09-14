---
title: "Crossplane XRD as Catalog Source"
space: UCP
parent_page_id: "../pocs.md"
---

# Crossplane XRD as Catalog Source

| | |
|---|---|
| **Related story** | [MCUCP-144](https://jira.rakuten-it.com/jira/browse/MCUCP-144) — Browse Service Catalog |
| **Status** | Complete — see [poc-report.md](crossplane-xrd-catalog-source/poc-report.md) |

## Goal

Verify whether the service catalog's source of truth can be Crossplane
`CompositeResourceDefinition` metadata instead of a hardcoded table — read cross-cluster
through a background polling cache — so that adding or removing a provider/service does not
require an api-server code change.

## Reading Order

1. [design.md](crossplane-xrd-catalog-source/design.md) — scope, hypothesis, approach, and
   success criteria — **start here**
2. [poc-report.md](crossplane-xrd-catalog-source/poc-report.md) — verdict and recommendation
3. [implementation.md](crossplane-xrd-catalog-source/implementation.md) — full proof detail

## Sub-docs

| Document | Role | Contents |
|----------|------|----------|
| [design.md](crossplane-xrd-catalog-source/design.md) | Human-first review | Question, hypothesis, scope, approach, success criteria, risks |
| [poc-report.md](crossplane-xrd-catalog-source/poc-report.md) | Human-first verdict | What was proven, what wasn't, recommendation |
| [implementation.md](crossplane-xrd-catalog-source/implementation.md) | Supporting proof | What is built, run, and verified |
