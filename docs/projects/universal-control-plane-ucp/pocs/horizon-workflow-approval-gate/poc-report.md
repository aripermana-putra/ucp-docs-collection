---
title: "Horizon Workflow Approval Gate PoC — Report"
space: UCP
parent_page_id: "../horizon-workflow-approval-gate.md"
---

# PoC Report: Horizon Workflow Approval Gate

**Research question answered:** [MCUCP-304](https://jira.rakuten-it.com/jira/browse/MCUCP-304) — Can Horizon Workflow serve as the backend for UCP's approval gate, with a multi-level approval chain dynamically scoped to the requesting tenant, and can UCP drive approvals from its own interface instead of requiring an OneCloud Portal login?

**Status:** Complete (Part A). Part B (resume callback) deferred, not attempted.

---

## Verdict

**Yes — Horizon Workflow can back a multi-level UCP approval gate, entirely self-service, and the resulting workflows are genuinely usable by a UCP-native interface.**

A two-layer approval chain — the submitting tenant's admins, then a fixed team — was built, published, and driven end to end against the QA environment, as both a test workflow and a real production workflow. Every mechanic UCP would need was proven with real API evidence: dynamic per-request approver resolution, a second independent approval layer, clean rejection short-circuiting, and visibility of the resulting production workflow through `GET /jobs`, the same API the OneCloud Portal's own Approvals tab is backed by.

## What This PoC Proved

| Criterion | Result |
|---|---|
| Kind + definition draft creation via self-service API, no Workflow-team involvement | PASS — no request to the platform team at any step |
| Layer 1 approver group resolves dynamically from the request's own `tenant_rns` parameter | PASS — resolved to the real, full list of 7 `clsd-ucp` admins, not just the operator |
| Layer 2 approver group resolves against a fixed Team, independent of the requesting tenant | PASS — resolved to `clsd-ucp-team`'s admin(s) |
| Rejecting at layer 1 prevents layer 2 from ever being reached | PASS — layer 2's step object was never instantiated (absent from state history entirely, not merely inaccessible) |
| Both layers approving reaches a `Successful` terminal outcome | PASS — for both test and production workflows |
| Test and production workflows behave identically for the approval-chain logic | PASS — same shape, same result, on both surfaces |
| A production workflow is visible via `GET /jobs?approver=<email>` / `?created_by=<email>` while open | PASS — same query surface returned the operator's own real, pre-existing approvals from other services (confirming this is the actual API behind the Portal's Approvals tab) |
| A UCP-native interface (CLI) could drive approvals without a Portal login | PASS (by construction) — `/jobs` is a plain REST resource, callable with a bearer token from the shared Keycloak realm UCP already uses |

**Prerequisites confirmed along the way**, easy to miss because a 403 gives no indication of which is missing:

- Tenant subscription to the Workflow service (a Core Data API call, not a Workflow API call).
- The Workflow `admin` role (Kind/definition management, test workflows) and, separately, the `creator` role (required specifically for production workflow creation).

**Operational gotchas confirmed**, relevant to any real integration:

- `process` (test workflows only) is valid once, right after creation — calling it again after a trigger returns `400 WF0050`, since there's nothing automated left to advance through.
- `execution_status` reads immediately after an action can transiently show `processing` before settling — a real integration must poll until stable.
- `/jobs` listing appears to default to in-progress workflows only, matching the Portal's own default filter — a completed workflow dropping out of `created_by`/`approver` listings is expected, not a visibility defect.

## What This PoC Did Not Prove

- **Whether UCP's own tenant/RBAC model maps 1:1 onto OneCloud tenant admins.** The PoC's tenant (`clsd-ucp`) is already a real OneCloud tenant, so its clean approver resolution does not generalize to UCP's actual multi-tenant case. If UCP's tenant model is independent, the built-in `tenant` user group kind won't resolve the right people, and a custom (HTTP-based) user group kind would be needed instead. This remains the single most important open question before designing a real Kind.
- **The resume callback (Part B).** Deferred entirely — every real callback URL found in the underlying research is an internal `r-local.net` service, and it was never confirmed whether Workflow's QA backend can reach anything a UCP-controlled endpoint would expose. No callback node was included in the tested definition at all.
- **The `cancel` action**, empirically. Its transition is structurally identical to `reject` (same shape, different trigger id and terminal outcome) and was not run separately — treated as low-risk by inference, not verified.
- **A drift-reconciliation-shaped trigger.** The tested Kind models a provisioning-request shape; no existing Workflow Kind anywhere in the platform models "resource drifted → reconciliation approval," so this specific trigger pattern remains unproven, though nothing in the state-machine model suggests it would behave differently.
- **Any real Temporal integration.** The PoC's script is a standalone driver, not a Temporal activity — how a real "resume a paused Temporal activity from an external signal" mechanism would work was not built or tested.

## Recommendation

Proceed with Horizon Workflow as a serious candidate for UCP's approval gate. Before committing to a real Kind design:

1. Resolve the tenant-mapping question with the Workflow Core Team — this decides whether the built-in `tenant` user group kind is usable as-is or whether a custom HTTP-based resolver is needed.
2. Treat Part B (the resume callback) as a separate, later decision — it is not required to validate the approval-chain mechanics or its visibility, both of which are now proven. If picked up, the reachability question needs an answer first (internal-network path from a UCP-owned service, most likely) before a callback node is worth building.
3. Decide the approver-facing surface as a product question, not a technical one: the shared OneCloud Portal dashboard, a UCP-native CLI/UI against `/jobs` directly, or both — all are now confirmed technically viable.

Full evidence, request/response traces, and the exact API contract corrections found along the way are in [Implementation](implementation.md). Scope, hypothesis, and what was deliberately left out are in [Design](design.md).
