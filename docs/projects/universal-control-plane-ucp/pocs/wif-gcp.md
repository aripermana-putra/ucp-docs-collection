---
title: "WIF — GCP Workload Identity Federation with Crossplane"
space: UCP
parent_page_id: "../pocs.md"
---

# WIF — GCP Workload Identity Federation with Crossplane

| | |
|---|---|
| **Ticket** | MCUCP-217 |
| **Parent research** | [Workload Identity Federation — Feasibility for UCP](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6761359577) |
| **Status** | Complete — `Secret` + `external_account` approach works end-to-end; GP 106 requires CCoE action for production |

---

## Goal

Verify whether `provider-upjet-gcp` supports Workload Identity Federation (WIF) for a self-hosted Crossplane running on a local (non-GKE) Kubernetes cluster, and identify which credential source works end-to-end.

---

## Reading Order

1. [design.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6761359559) — scope, hypothesis, approach, and success criteria — **start here**
2. [poc-report.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6761359552) — verdict after the PoC completes
3. [concepts.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6760478998), [implementation.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6761359561), [evidence.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6759410640) — supporting detail

---

## Sub-docs

| Document | Role | Contents |
|----------|------|----------|
| [design.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6761359559) | Human-first review | Question, hypothesis, scope, approach, success criteria, risks |
| [poc-report.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6761359552) | Human-first verdict | What the PoC proved and the recommendation |
| [concepts.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6760478998) | Supporting context | How WIF works, credential source options, prerequisites |
| [implementation.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6761359561) | Supporting proof | What was built, run, and observed |
| [evidence.md](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6759410640) | Supporting evidence | Provider source code findings, external references |
