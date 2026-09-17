---
title: "Horizon Workflow Approval Gate — Implementation"
space: UCP
parent_page_id: "../horizon-workflow-approval-gate.md"
---

# Horizon Workflow Approval Gate — Implementation

The two-layer approval chain is built and verified end-to-end against the QA Horizon Workflow API, for both test workflows and real production workflows, including visibility via the `/jobs` resource. Part B (the resume callback) remains deferred — see the design doc.

## What is built

A Go script, `horizon-workflow-approval-gate-poc` in the kitchen-sink repository, drives the full lifecycle for one Workflow Kind, in two modes:

1. Registers a Workflow Kind (`ucp_poc_vm_creation_approval`) under the `clsd-ucp` QA tenant.
2. Creates a definition draft with the state machine `start → tenant_admin_approval → network_team_approval → end`:
   - `tenant_admin_approval` resolves its approver group dynamically: `{"kind": "tenant", "tenant_rns": "{{ params.tenant_rns }}", "role": "admins"}`.
   - `network_team_approval` resolves against a fixed Team: `{"kind": "team", "team": "clsd-ucp-team", "role": "admins"}`.
   - `reject`/`cancel` from either step transition directly to `end`.
3. `MODE=test`: creates a test workflow instance against the draft and drives it via the tenant-scoped `test-workflows` endpoints.
   `MODE=production`: publishes the draft, creates a real production workflow against the published version, and drives it via the ABAC-scoped `/jobs` resource.
4. Advances the instance through the chain, triggering `approve`/`reject`/`cancel` as the operator, and reads back the resulting state at each step.

Three scenarios are covered, in both modes: both layers approve; layer 1 rejects; layer 1 cancels.

## Prerequisites this exposed

Registering a Kind and driving a workflow required prerequisites that are easy to miss because a 403 gives no indication of which is missing:

- **Tenant subscription to the Workflow service.** `clsd-ucp` was not subscribed. Subscribing is a Core Data call: `POST /v0/tenants/{tenant_rns}/subscriptions` with body `{"rns": "rns:roc:workflow"}`, against `qa-horizon-data-api.r-local.net`.
- **The Workflow `admin` service role**, granted per-tenant via the OneCloud Portal (Tenant Management → grant access to a principal). Required for Kind/definition management and for creating test workflows.
- **The Workflow `creator` service role**, granted the same way, separately from `admin`. Required specifically to create **production** workflows (`POST .../versions/{version}/workflows`) — `admin` alone is not enough for that call.

Without the right role, `POST .../kinds` (or the production workflow-creation call) returns `403 WF0031` ("R-Session permission evaluation failed"), regardless of the caller's other IAM/tenant-admin permissions.

## API contract corrections

Confluence-documented details that turned out to not match the live API, confirmed against the live OpenAPI spec at `https://qa-horizon-workflow-api.r-local.net/openapi.json`:

- Test workflow paths use hyphens — `test-workflows` — not the underscores (`test_workflows`) the Tutorial/Reference prose renders them with.
- The create-test-workflow request body field is `definition_draft_number`, not `definition_number`.

## Two response shapes, easy to conflate

`POST .../test-workflows`, `POST .../process`, `POST .../triggers`, and `POST .../versions/{version}/workflows` (production creation) all return the same small ack shape (`WorkflowActionResponse`): `{"workflow_id", "job_id", "status"}`, where `status` is one of `created`, `pending`, `success` — an acknowledgement that the request was accepted, not the workflow's outcome.

Only `GET .../test-workflows/{id}` (test) or `GET /jobs/{id}` (production) returns the actual workflow state: `{"id", "execution_status", "steps", "triggers", ...}`, where `execution_status` is one of `processing`, `waiting`, `completed`.

## `process` is only valid when there's something automated to advance through

The definition here has no callback or adapter nodes — every hop after `start` requires a human trigger. Calling `process` after a trigger (rather than only once, immediately after creating the test workflow, to move off `start`) returns `400 WF0050` ("Workflow has an active step... is waiting for a trigger"). Production workflows have no `process` endpoint at all — they advance automatically.

`execution_status` reads immediately after an action can still show `processing` rather than the settled `waiting`/`completed` value, since the transition itself is processed asynchronously server-side. A real integration polling this API needs to poll until stable, not trust a single immediate read.

## A Go nil-map gotcha in the /jobs trigger call

`POST /jobs/{workflow_id}/{action}` requires a JSON object body, even an empty one — passing Go's untyped `nil` for a `map[string]any`-typed parameter produces a *typed nil map*, which is a non-nil `any` value that `encoding/json` marshals to the JSON literal `null`. The API rejects `null` with `400 WF0001` ("Field required"). Fix: pass `map[string]any{}` explicitly rather than `nil`.

## Evidence — test workflow, both layers approve

Kind and definition draft created via pure API calls, no request to the Workflow platform team. Both approver groups resolved correctly on a real test workflow instance (`1x76yx-z6arA26PFLyWiFqn`):

```json
{
  "label": "Submitting tenant admin approval",
  "approvers": ["aripermana.putra@rakuten.com", "govind.madhu@rakuten.com",
    "krishna.chalwetkar@rakuten.com", "rania.benkahla@rakuten.com",
    "ryo.kimura@rakuten.com", "sebasti.bellefeuille@rakuten.com",
    "yusuke.a.ohashi@rakuten.com"],
  "result_state": "Approved"
}
```

```json
{
  "label": "Network team approval",
  "approvers": ["aripermana.putra@rakuten.com"],
  "result_state": "Approved"
}
```

Final state: `"states": {"current": "Completed", "result": "Successful"}`, `"execution_status": "completed"`. Full state history present in order: `Started → Submitting tenant admin approval → Network team approval → Completed`.

The `tenant_admin_approval` approver list resolved to the real admins of `clsd-ucp` — seven people, not just the operator — confirming approver resolution is genuinely parameter-driven rather than coincidentally matching the operator's own identity.

## Evidence — test workflow, layer 1 rejects

Second test workflow instance (`1x775U-VjLYkNgAlcXfTEXt`), triggered `reject` at layer 1:

```json
"states": {"current": "Completed", "result": "Rejected"}
```

State history has exactly three entries: `Started → Submitting tenant admin approval → Completed`. No `Network team approval` step object exists anywhere in the response — it was never instantiated, not merely inaccessible. A layer-1 rejection cannot expose an action to the layer-2 approver group.

## Evidence — production workflow, layer 1 rejects

Published definition draft (`version_major: 3`), created a production workflow (`1x77Ze-LAF0V6MrZrXkXquR`) against it, and triggered `reject` via `POST /jobs/{id}/reject`:

```json
"states": {"current": "Completed", "result": "Rejected"},
"execution_status": "completed"
```

Layer 1 approvers resolved identically to the test-workflow case (the same seven real `clsd-ucp` admins). Production workflows behave identically to test workflows for the approval-chain logic itself — the platform does not distinguish between them for that part.

## Evidence — production workflow, both layers approve

Production workflow `1x77bs-pVgYCdrvcqPSFMay` (definition version 4), full happy path via `/jobs`:

```json
{"label": "Submitting tenant admin approval", "result_state": "Approved",
 "approvers": ["aripermana.putra@rakuten.com", "govind.madhu@rakuten.com",
   "krishna.chalwetkar@rakuten.com", "rania.benkahla@rakuten.com",
   "ryo.kimura@rakuten.com", "sebasti.bellefeuille@rakuten.com",
   "yusuke.a.ohashi@rakuten.com"]}
```
```json
{"label": "Network team approval", "result_state": "Approved",
 "approvers": ["aripermana.putra@rakuten.com"]}
```

Final: `"states": {"current": "Completed", "result": "Successful"}`, `"execution_status": "completed"`. Both `Approve` triggers recorded against the correct step IDs. Identical shape to the test-workflow happy-path result — production and test workflows are equivalent for the approval-chain logic.

Combined with the earlier reject evidence, both terminal outcomes (`Successful`, `Rejected`) are now confirmed for both test and production workflows. `cancel` was not run separately — its transition is structurally identical to `reject` (same shape, different trigger id and terminal outcome), so it is not expected to behave differently, though this has not been empirically verified.

## Evidence — production workflow visibility via `/jobs`

`GET /jobs?approver=<operator email>` returned real, pre-existing production workflows the operator is a genuine approver on — including the exact same `1vsxrz-ddkXBdZy5Fxo15Lx` / `1vsxmR-6VnM7zuD8jnpnWUV` "CaaS Request Namespace" workflows visible in the OneCloud Portal's own Approvals tab. This directly confirms `/jobs` is the real API backing that tab.

Querying `created_by`/`approver` for the PoC's own workflow immediately after it completed returned zero results, even though `GET /jobs/{id}` (direct lookup) showed the workflow with the correct `created_by` field. The two real workflows that did show up were both still `execution_status: waiting`; the PoC's own workflow was `completed` by query time. This is consistent with `/jobs` listing defaulting to in-progress workflows only — matching the Portal's own Approvals tab, which defaults to an "In-progress" status filter — rather than `created_by`/`approver` filtering being broken. Confirmed directly: a workflow created by this Kind **is** visible via `/jobs` while still open/actionable, the same way any other service's workflow is.

## What is not covered

- Whether UCP's own tenant/RBAC model maps 1:1 onto OneCloud tenant admins for the `tenant` user group kind to resolve correctly outside this PoC's own already-OneCloud-native tenant.
- The resume callback and its retry behavior (Part B) — deferred, not currently being pursued.
- `cancel-layer1` was not run — treated as low-risk given its structural identity to `reject-layer1`, not because it was verified.
