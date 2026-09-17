---
title: "Horizon Workflow Approval Gate"
space: UCP
parent_page_id: "../pocs.md"
---

# Horizon Workflow Approval Gate

Validates whether Horizon Workflow (OneCloud's WaaS platform) can back UCP's approval
gate for provisioning and drift-reconciliation requests, with approvers dynamically
scoped to the requesting tenant's admins via a parameter-driven user group, and a
second approval layer scoped to a fixed team. Confirmed viable — see the report.

Parent investigation: [OneCloud Horizon Workflow Service as UCP's Approval Gate — Feasibility Research](../research/onecloud-horizon-workflow-service.md)

## Sub-Documents

- [PoC Report](horizon-workflow-approval-gate/poc-report.md) — verdict, what was proved and not proved, recommendation
- [Design](horizon-workflow-approval-gate/design.md) — question, hypothesis, scope, approach, success criteria, risks
- [Implementation](horizon-workflow-approval-gate/implementation.md) — what was built, prerequisites, API contract corrections, evidence from every scenario
