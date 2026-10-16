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

## Evidence — conditional network-team step

The network-team step carries `skip_if` on `params.vlan`, a payload-less `skip` trigger named by `skip_with`, and a `complete` trigger whose payload requires `vlan`. Runs via `VARIANT=vlan`:

| Creation payload | `skip_if` | Result |
|---|---|---|
| `vlan` absent | `{{ params.vlan is defined and params.vlan }}` | Step waits; `complete` with `vlan` ends `Successful`; the `complete` trigger is recorded in the workflow |
| `vlan: "vlan-200"` | `{{ params.vlan is defined and params.vlan }}` | `Error` outcome, step not marked skipped, note "An unexpected error arose, contact the Workflow support team" (node: Network team approval) |
| `vlan: "vlan-200"` | `{{ params.vlan is defined and params.vlan \| length > 0 }}` | Step shown as "Network team approval (skipped - VLAN provided)", `is_skipped: true`, result `Skip`; workflow ends `Successful` after one approval |

`skip_if` is typed as boolean-or-Jinja. Jinja's `a and b` returns `b` when `a` is true, so the first expression yields the string `vlan-200`, which produces the generic `Error`. The same generic note appeared for this expression fault, so it does not identify the cause. Using a separate payload-less trigger for `skip_with` was tried while diagnosing and did not change the first failure; whether `skip_with` can name the payload-requiring `complete` trigger once `skip_if` is a boolean was not tested. The supplied `vlan` is readable as `params.vlan`. Copying the `complete` payload into workflow `data` with a `data_update` adapter was not tested.

## Evidence — first-layer approver from a UCP service role

Production workflows with `service_resource_role` (`service_rns` `rns:roc:ucp`, role `test-admin`), `VARIANT=service-role`:

| `resource_rns` | Role | Resolved `approve` groups | Operator's `approve` |
|---|---|---|---|
| `""` | `test-admin` | none (group dropped as null) | allowed |
| `""` | `does-not-exist` | none (group dropped as null) | allowed |
| `tenants:clsd-ucp` | `test-admin` | one group, title "ROC service ucp's resource tenants:clsd-ucp test-admins", zero emails | rejected: `WF0010`, allowed triggers `["cancel"]` |

The definition validates (`definitions:validate`) with and without a resource. The role name is not checked when the resource is empty. UCP defines no Core Data resources, and the operator's `test-admin` is a coarse-grained grant, so a non-empty resource resolves to nobody. With an empty resource, the step has no approver groups and the operator, who is the requester and holds the Workflow `admin` and `creator` roles, can still approve.

## Evidence — first-layer approver from a list of emails

`VARIANT=user-groups` builds five `user` groups reading `params.approver_emails[i]`, each evaluating to an empty value when the list is shorter, with the first carrying a `backup_spec` to `clsd-ucp-team`:

| `approver_emails` | Resolved `approve` groups | Operator's `approve` |
|---|---|---|
| the operator | one group with that email (other slots dropped) | allowed; `Successful` |
| empty | none; the backup team is not present | allowed; `Successful` |

An empty list leaves the step without approver groups, and the backup group is not applied to a null group. Who, other than the operator, could approve in the empty case was not tested; this needs a requester who holds neither the `admin` nor the `creator` role.

## Evidence — custom notification email

`VARIANT=email` adds an `email` callback node before the first approval step (`recipient` `{{ params.approver_emails[0] }}`, subject and body over `params`). The node reports `success` and the workflow proceeds; the message arrives in the recipient's inbox from the QA Workflow sender with the rendered request details. Two further production workflows (`SCENARIO=reject-layer1`, distinct `RESOURCE_ID` in the custom subject so each custom email identifies its workflow) compared the built-in emails:

| Run | Workflow | Approve path ends in | Built-in emails received |
|---|---|---|---|
| control (`VARIANT=email`) | `1xEGAQ-CGTy8X2rhkk48LI4` | `Successful` | created, ask for review, updated, rejected |
| `VARIANT=email,no-successful` | `1xEGAr-b5espkXsL7MJcej8` | `Completed` | created, updated, rejected |

Both runs also delivered the custom email. With no path setting `Successful`, the built-in "ask for review" email to the step's approvers was not sent; the other built-in emails were. This matches the Explanation page's rule that a step email goes only to users who can take a required action. Each variant was run once, with the requester and the approver being the same person. Not tested: whether such approvers still see the workflow in the Portal's Approvals tab or `GET /jobs?approver=`, and whether the workflow's outcome is usable by the caller when it can never be `Successful` (approving ends as `Completed`).

## Evidence — custom user group kind over Core Data

A custom `http` user group kind (`tenant-ucp-admins`) reads a tenant's members from the Core Data API with `oauth2_client_credentials`, using a UCP service client (`rns:roc:ucp-ops`) whose secret and the Keycloak token URL are stored as a tenant secret and tenant config in the Workflow tenant. A step uses it as `{"kind": "tenant-ucp-admins", "tenant_rns": <owner tenant>, "tenant": "{{ params.tenant_name }}"}`. The log is `run-20261015-*-custom-kind-oauth.log` in the kitchen-sink PoC directory (no secrets in it).

- The client obtains a token (600 s) from the QA Keycloak and reads the members of `clsd-ucp`, `clsd-ucp-qa4` and `iam` (HTTP 200 each), although its own groups are all `clsd-ucp` roles.
- The processor must follow the dialect the registration endpoint validates. Accepted: filters after a concrete path, chained filters, and a single `&` for AND. Rejected: filters after `..`, `&&`, and `and`.
- Role filtering is exercised against `clsd-ucp-qa4`, which has a developer-only and a viewer-only UCP member: the `tenant-admin` processors return 9 emails and exclude both. A service-only filter returns every UCP member regardless of role.
- Two production workflows from one published definition, differing only in `params.tenant_name`: `clsd-ucp` resolved 5 approvers and `clsd-ucp-qa4` resolved 9, each matching the members with the UCP `tenant-admin` role. `approve`, `reject` and `cancel` were allowed for the operator, who is among the resolved admins.
- Static `Authorization` headers are not used; the credential is obtained by Workflow from the client id and secret.
- The kind, secret, config and both workflows were removed afterwards.

The same kind with the processor filtering on the UCP role `test-admin` (held only by the operator in `clsd-ucp`) gives:

| Tenant | `users/sync` | Workflow first step |
|---|---|---|
| `clsd-ucp` | the operator's email only | one approver; `approve`, `reject`, `cancel` allowed |
| `clsd-ucp-qa4` (no holder) | `400 WF0062` | step created with a group that resolves to nobody; the operator's `approve` and `reject` are refused (`allowed: false`), only `cancel` is allowed |
| `iam` (no holder) | `400 WF0062` | not run as a workflow |

`WF0062` is therefore returned both for a failed call and for a processor that matches nobody. Unlike an empty `user` group or an empty `resource_rns`, which are dropped as null groups, a custom kind group that resolves to nobody is kept and no one can approve. A rerun of the `clsd-ucp-qa4` case (`run-20261015-*-custom-kind-test-admin-empty-tenant.log`) read the workflow after its step existed: `triggers_users.approve` and `.reject` each hold the custom group with `emails: []`, while `cancel` holds only the requester; the operator is allowed `cancel` only. The other logs are `run-20261015-*-custom-kind-test-admin*.log` (the first ends with a script error, the second removes the kind, secret, config and workflows).

Not covered: the production `rns:roc:ucp` client and its permissions across all tenants UCP serves; behavior when the secret or config key is missing; refresh of membership on an open workflow.

## Evidence — fingerprint

`VARIANT=fingerprint` (production mode) publishes a definition whose top-level `fingerprint_fields` is `["resource_type", "resource_id", "tenant_name"]`; `payload_schema` also carries `reason`, which is not a fingerprint field. The run (`run-20261016-095039-fingerprint.log`) creates workflows in this order, all by the same requester against the same published version:

| Case | Payload | Result |
|---|---|---|
| B | base values, `reason` set | created, stays open |
| C | identical to B while B is active | `409 WF0011` "An active workflow with the same fingerprint exists"; the body carries the blocking `workflow_id` and the fingerprint values |
| E | `resource_id` suffixed `-other` | created |
| F | `tenant_name` removed | `400 WF0040` "Missing part of the fingerprint"; the body lists `fingerprint_fields` |
| D | identical to B after B is cancelled | created |

The duplicate check therefore applies only to active workflows; it did not block a repeat once the first was cancelled. A control run without a fingerprint is in `run-20261016-095055-fingerprint-control.log`.

The "Your review is required" email for workflow E (screenshot from the operator's inbox) lists, after the workflow type: `Resource Type` (`vm`), `Resource Id`, `Tenant Name`, the approver's role(s), the requester and the request time, followed by a "What is this workflow?" box holding the Kind's description and a "View full workflow details" button. The fingerprint fields are labeled by their field name in title case. `reason`, which is not in the fingerprint, does not appear in the email, although the workflow's page in the OneCloud Portal shows it. The email also lists the actions the approver may take (Approve, Cancel, Reject).

A second run (`VARIANT=fingerprint,user-groups`, approver list the operator only; `run-20261016-100250-fingerprint-version-race.log`) repeated B to F and added two cases:

| Case | Setup | Result |
|---|---|---|
| G | workflow G1 open on version 25; the same definition is published as version 26; the same payload is created on version 26 | `409 WF0011` naming G1 (version 25) as the blocking workflow |
| H | five creations of one new payload issued at the same instant on version 26 | all five created, five distinct workflows |

The check therefore spans published versions of the Kind, and it does not hold under simultaneous creations: concurrent requests for one fingerprint can each pass. All workflows from the run were cancelled and read back as `Canceled`/`completed`. A cancel issued right after creation is refused with `WF0010` until the first step exists; the script retries it.

A third run (`VARIANT=fingerprint-race,user-groups`, approver list the operator only; `run-20261016-101157-fingerprint-race-window.log`) measured the width of the window. Each row is three pairs of creations of one new payload, the second creation delayed by the stated time:

| Delay of the second creation | Accepted per pair (3 pairs) |
|---|---|
| 0 ms | 2, 2, 2 |
| 100 ms | 2, 2, 2 |
| 300 ms | 2, 2, 2 |
| 1000 ms | 1, 1, 1 (the second got the `409`) |

The window is therefore longer than 300 ms and shorter than 1000 ms in this environment; the delay is measured from when the second request is issued, so request latency is included. The 21 accepted workflows were cancelled and read back as `Canceled`/`completed`.

A fourth run (`VARIANT=fingerprint-types,user-groups`, approver list the operator only; `run-20261016-105450-fingerprint-types.log`) added six fingerprint fields of other types and created one workflow with them, then repeated the payload. The `409 WF0011` body returns every value with its JSON type intact (number, boolean, object, array, the full 630-character string, the HTML string unchanged). The review email lists each field in the order of `fingerprint_fields`, labeled by the field name in title case with underscores as spaces:

| Field | Value sent | Shown in the email |
|---|---|---|
| `dry_run` (boolean) | `true` | `True` |
| `labels` (object) | `{"env":"qa","team":"ucp"}` | `{ env: qa, team: ucp }` |
| `markup` (string) | `<b>bold</b> & "quoted" 'single' <script>x</script>` | the same text, escaped and not rendered |
| `note` (string, 630 characters) | `long-note-segment-` repeated | cut at about 120 characters and ended with `...` |
| `quota` (number) | `12.5` | `12.5` |
| `tags` (array) | `["alpha","beta",3]` | `[ alpha, beta, 3 ]` |

Not covered: the other built-in emails (created, canceled) for the fingerprint workflows and the control run; delays between 300 and 1000 ms, and whether the window changes with load.

## Evidence — idempotency inputs

`VARIANT=idempotency,user-groups` (production mode, approver list the operator only; `run-20261016-101741-idempotency.log`) probes the two candidate inputs on version 28 of the Kind, with the fingerprint from the previous section in place.

`X-Request-ID`, one fresh UUID `U`, creations issued two seconds apart:

| Case | Header | Payload | Result |
|---|---|---|---|
| I1 | `U` | P1 | created |
| I2 | `U` | P2 (different `resource_id`) | created, a new workflow with a different id |
| I3 | `U` | P1 again | `409 WF0011` (fingerprint) |
| I4 | none | P3 | created |
| I5 | none | P3 again | `409 WF0011` (fingerprint) |

The header does not deduplicate: the same `U` created two workflows for different payloads and did not return the first workflow for the repeated payload; I3 and I5 behave identically, so the `409` comes from the fingerprint alone.

`POST /jobs:check-duplicate` (`workflow_kind`, `version`, `parameters`) returns `{"is_duplicate": bool}`:

| Call | `is_duplicate` |
|---|---|
| active payload P3 | `true` |
| unknown payload P4 | `false` |
| issued 0 ms after the creation of the same payload (2 trials) | `false`, `false` |
| 300 ms (2 trials) | `false`, `false` |
| 600 ms (2 trials) | `true`, `true` |
| 1000 ms and 1500 ms (2 trials each) | `true` |

A workflow only counts as a duplicate for `check-duplicate` from somewhere between 300 and 600 ms after its creation request, which matches the race window seen for creation (accepted at 300 ms, refused at 1000 ms). `check-duplicate` is therefore a read of the same state and does not close the window. The 13 workflows from the run were cancelled and read back as `Canceled`/`completed`.

Not covered: whether `check-duplicate` without `version` spans versions; the exact boundary between 300 and 600 ms.

## What is not covered

- Whether UCP's own tenant/RBAC model maps 1:1 onto OneCloud tenant admins for the `tenant` user group kind to resolve correctly outside this PoC's own already-OneCloud-native tenant.
- The resume callback and its retry behavior (Part B) — deferred, not currently being pursued.
- Step expiry: no mechanism was found in the API schema or the Reference page.
- `cancel-layer1` was not run — treated as low-risk given its structural identity to `reject-layer1`, not because it was verified.
