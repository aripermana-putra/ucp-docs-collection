---
title: "Crossplane XRD Versioned Schema and Provisioning"
space: UCP
parent_page_id: "../pocs.md"
---

# Crossplane XRD Versioned Schema and Provisioning

| | |
|---|---|
| **Related stories** | [MCUCP-145](https://jira.rakuten-it.com/jira/browse/MCUCP-145) — Service Detail View, [MCUCP-146](https://jira.rakuten-it.com/jira/browse/MCUCP-146) — Provision Catalog Entry |
| **Status** | Complete — see [poc-report.md](crossplane-xrd-versioned-schema-provisioning/poc-report.md) |

## Goal

Verify whether a per-served-version JSON Schema derived from a `CompositeResourceDefinition`
can drive `describe`/`generate-template`-equivalent reads and `provision`-equivalent
validate-then-apply writes, entirely from the XRD's own schema — with no availability check
and no real provisioning backend. Builds on
[crossplane-xrd-catalog-source](crossplane-xrd-catalog-source.md), which proved the catalog
*metadata* derivation path; this PoC covers the *schema* derivation and provisioning-submission
path instead.

## Reading Order

1. [design.md](crossplane-xrd-versioned-schema-provisioning/design.md) — scope, hypothesis,
   approach, and success criteria — **start here**
2. [poc-report.md](crossplane-xrd-versioned-schema-provisioning/poc-report.md) — verdict and
   recommendation
3. [implementation.md](crossplane-xrd-versioned-schema-provisioning/implementation.md) — full
   proof detail

## Sub-docs

| Document | Role | Contents |
|----------|------|----------|
| [design.md](crossplane-xrd-versioned-schema-provisioning/design.md) | Human-first review | Question, hypothesis, scope, approach, success criteria, risks |
| [poc-report.md](crossplane-xrd-versioned-schema-provisioning/poc-report.md) | Human-first verdict | What was proven, what wasn't, recommendation |
| [implementation.md](crossplane-xrd-versioned-schema-provisioning/implementation.md) | Supporting proof | What is built, run, and verified |
