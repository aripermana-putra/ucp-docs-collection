---
title: "Horizon Workflow Approval Gate — Design"
space: UCP
parent_page_id: "../horizon-workflow-approval-gate.md"
---

# Horizon Workflow Approval Gate — Design

## Question

Can Horizon Workflow (OneCloud's WaaS platform) serve as the backend for UCP's approval gate — the step between a tenant's provisioning/drift-reconciliation request and the actual action — with approvers dynamically scoped to the requesting tenant's admins, and an approval surface UCP can drive from its own interface (e.g. a CLI) rather than requiring users to log into the OneCloud Portal?

This spike has two parts. Part A validates the approval chain itself, against test workflows and then against real production workflows. Part B (the resume callback) is deferred indefinitely — see Risks — since the approval chain and its visibility to UCP matter more immediately than the resume mechanism.

## Hypothesis

**Part A** — A Workflow Kind, defined with a Start → Step (approval) → Step (approval) → End state machine, can:

1. Be created and driven end-to-end via the Workflow API without requiring anything from the Workflow platform team (no legacy kind, no transfer request).
2. Resolve its first-layer approver group dynamically per request, using the built-in `tenant` user group kind with `tenant_rns` bound to a workflow parameter, rather than a hardcoded team.
3. Resolve a second, independent approval layer against a fixed Team (`clsd-ucp-team`, standing in for a real network-team group), using the built-in `team` user group kind — modeling a real multi-level approval case (e.g. a VM creation request needing both the submitting tenant's admin and a network team's admin).
4. Reject or cancel cleanly at either layer — layer 2 must never be reached, and the outcome must reflect which layer stopped it.
5. Once published, be visible and actionable as a **production** workflow via the ABAC-scoped `/jobs` resource — the same resource backing the OneCloud Portal's Approvals/My Requests tabs — so that a UCP-native interface can call it directly, using a bearer token from the shared Keycloak realm, without requiring a Portal login.

**Part B** (deferred, not currently pursued) — Once layer 2 approves, a Callback node could deliver to an externally reachable endpoint, with retry behavior a UCP-side handler can absorb idempotently.

## Scope

In scope:

- QA environment only (`qa-horizon-workflow-api.r-local.net`, QA OneCloud Portal).
- One Workflow Kind, one definition draft — `start → tenant_admin_approval → network_team_approval → end`, no callback node.
- A minimal script (not a service) that drives the chain two ways:
  - As a **test workflow**, via the tenant-scoped `test-workflows` endpoints — no publishing needed, impersonation available, never visible in the Portal.
  - As a **production workflow**: publish the definition draft, create a production workflow against the published version, and drive it via `POST /jobs/{workflow_id}/{action}` and `GET /jobs/{workflow_id}` — the ABAC-scoped resource a real UCP interface would call.
- Three scenarios, run against both test and production workflows: both layers approve; layer 1 rejects; layer 1 cancels. In every case the operator acts directly (they are admin of both the submitting tenant and `clsd-ucp-team` — no impersonation needed).
- A visibility check: `GET /jobs?created_by=<operator>` and `GET /jobs?approver=<operator>`, confirming the production workflow is queryable the same way the Portal's tabs would show it, without going through the Portal.

Deferred to Part B (not attempted, no active plan to attempt):

- The Callback node, and verifying Workflow's retry behavior on a failed delivery.
- Finding or building a way for Workflow's QA backend to reach a receiver this PoC controls — see Risks.

Out of scope entirely:

- Resolving whether UCP's own (independent) tenant/RBAC model maps 1:1 onto OneCloud tenant admins — this PoC's own tenant is already a real OneCloud tenant, so it cannot by itself prove or disprove that mapping for UCP's actual multi-tenant case. That remains a separate open question (see Research doc).
- Any Temporal integration.
- Building UCP's own CLI/interface for real — this PoC only confirms the API surface such an interface would call.

## Approach

1. Confirm QA tenant subscription to the Workflow service, and grant the service account **both** the `admin` role (Kind/definition management, and required for test workflow creation) and the `creator` role (required separately for creating **production** workflows — `admin` alone does not grant that). No separate setup is needed for layer 2 — `clsd-ucp-team` is already a real QA Team the operator is Admin of.
2. Register a Workflow Kind (e.g. `ucp_poc_vm_creation_approval`) under the QA tenant via `POST /workflow-sapi/2/tenants/{tenant_rns}/kinds`.
3. Author a definition draft with two sequential approval steps and no callback node:
   - `start → tenant_admin_approval (step) → network_team_approval (step) → end`
   - `tenant_admin_approval`'s `users` set to `{"kind": "tenant", "tenant_rns": "{{ params.tenant_rns }}", "role": "admins"}` — dynamic, resolved from the request's own `tenant_rns` param
   - `network_team_approval`'s `users` set to `{"kind": "team", "team": "clsd-ucp-team", "role": "admins"}` — hardcoded, always resolves to `clsd-ucp-team` regardless of which tenant submitted the request
   - `network_team_approval`'s `approve` transition goes directly to `end` with outcome `Successful`
   - `reject`/`cancel` transitions from either step route directly to `end` with outcome `Rejected`/`Canceled`, without reaching the other step
4. Against a **test workflow**: create an instance with a `tenant_rns` parameter pointing at the QA tenant, and run each of the three scenarios (approve both, reject at layer 1, cancel at layer 1), confirming the resulting terminal state each time.
5. Against a **production workflow**: publish the definition draft, create a production workflow instance against the published version, and re-run the same three scenarios via `/jobs` instead of `test-workflows`. After a successful run, confirm the workflow is queryable via `GET /jobs?created_by=<operator>` and `GET /jobs?approver=<operator>`.
6. Part B, if ever picked back up: add the callback node back, resolve the reachability question in Risks, and re-run the happy path to observe delivery and retry behavior.

## Success criteria

Test workflows:

- A Workflow Kind and definition draft are created purely through self-service API calls, with no request to the Workflow platform team.
- Layer 1 approver resolution is provably parameter-driven; layer 2 is provably scoped to `clsd-ucp-team` regardless of which tenant submitted the request.
- Reject and cancel at layer 1 both terminate cleanly without layer 2 ever being reached; approving both layers reaches `Successful`.

Production workflows:

- The definition draft publishes successfully and a production workflow can be created against the published version, using the `creator` role.
- The same approver resolution and reject/cancel/approve behavior holds for production workflows as it did for test workflows.
- The production workflow is visible via `GET /jobs` filtered by `created_by` and by `approver` — confirming a UCP-native interface could list and act on it without the OneCloud Portal.

Part B (deferred, not being pursued currently):

- The callback reaches a receiver with the expected payload only after both layers approve, and a deliberately-failed delivery is retried by Workflow without manual intervention.

## Risks

- **Callback reachability is unresolved** — every real callback URL found in the research is an internal `r-local.net` service, not a public one, so a laptop script has no obvious way to receive one. This is why Part B is deferred rather than actively blocked-and-waiting: there's no current plan to resolve it, since the approval-chain and visibility questions (Part A) don't depend on it at all.
- **Scope leakage**: it would be easy to drift into building a full Temporal integration, or a real UCP CLI, during this spike. Both remain explicitly out of scope — this PoC only needs to confirm the API surface exists and behaves as expected.
- **Non-representative tenant mapping**: because the PoC's tenant is already a real OneCloud tenant, a successful approver resolution here does not generalize to confirm UCP's actual (possibly independent) tenant model will resolve correctly — that requires the separate conversation with the Workflow team noted in the research doc.

## Open questions

- Same open question as the parent research doc: does UCP's tenant model need a custom user group kind, or does the built-in `tenant` kind suffice? This PoC cannot answer that on its own since it uses an already-OneCloud-native tenant.
- Does `GET /jobs` reliably show a workflow to every intended viewer (requester, current-step approvers, viewers), or only some of those roles? Only `created_by` and `approver` filters are exercised here.
