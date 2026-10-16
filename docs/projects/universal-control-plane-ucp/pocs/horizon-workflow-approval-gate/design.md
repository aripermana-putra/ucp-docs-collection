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
6. Make the second layer conditional on data supplied at creation. The `POST .../versions/{version}/workflows` body's free-form `payload` is exposed to the definition as `params`, so the network-team step can carry `skip_if: "{{ params.vlan is defined and params.vlan | length > 0 }}"` (the expression must evaluate to a boolean) with a `skip_with` trigger name. A request that supplies `vlan` bypasses the network-team step; a request without it waits for the network team, whose completion trigger carries the `vlan` value in its payload.
7. Resolve the first-layer approver group from a list of emails supplied at creation, instead of a OneCloud tenant. The creator looks up the holders of the UCP `tenant-admin` role in Core Data and passes them as `params.approver_emails`. The first step's triggers carry a fixed number of `user` groups, one per slot, each reading `params.approver_emails[i]` and evaluating to an empty value (a null group, removed) when the list is shorter. The first slot carries a `backup_spec` to a fixed team, so an empty list does not leave the step without approvers. `service_resource_role` with no `resource_rns` is also exercised, to record how it resolves.
8. Deliver a custom notification carrying the request details. The creator supplies the approvers' emails in the creation `payload` (`params.approver_emails`), and an `email` callback node before the first approval step sends the message. The node's `subject` and `body` are Jinja over `params`.
9. Suppress the platform's automatic step notification. A step's email goes only to users who can take a *required* action, meaning one that leads to at least one path setting the `Successful` outcome. If no path from the step sets `Successful`, the step's users have no required action and may receive no automatic email. This is unproven.
10. Resolve the first-layer approvers through a custom user group kind that reads a tenant's members from the Core Data API using a UCP-owned service client (`client_credentials`), with the client secret and the Keycloak token URL stored as tenant secret and config in the Workflow tenant. Confirms that the client can read other tenants' members and that Workflow resolves the extracted emails.
11. Show request details in the built-in approval email and block duplicate requests through the definition's `fingerprint_fields` (a nullable array of strings at the top level of the definition, next to `payload_schema`). The Workflow team indicated that the fields are listed in the "Your review is required" email. A second active workflow with the same field values is rejected with `409 WF0011`, and a creation payload that lacks a fingerprint field is rejected with `WF0040`. None of this is documented.
12. Find out whether a caller-supplied identifier can make workflow creation idempotent. The creation payload has no such field. Every endpoint accepts an optional `X-Request-ID` header (a UUID, generated when absent, described as a request identifier), and `POST /jobs:check-duplicate` (`workflow_kind`, optional `version`, `parameters`) returns `is_duplicate`. Neither is documented as an idempotency mechanism.

**Part B** (deferred, not currently pursued) — Once layer 2 approves, a Callback node could deliver to an externally reachable endpoint, with retry behavior a UCP-side handler can absorb idempotently.

## Scope

In scope:

- QA environment only (`qa-horizon-workflow-api.r-local.net`, QA OneCloud Portal).
- One Workflow Kind, one definition draft — `start → tenant_admin_approval → network_team_approval → end`, no callback node.
- A minimal script (not a service) that drives the chain two ways:
  - As a **test workflow**, via the tenant-scoped `test-workflows` endpoints — no publishing needed, impersonation available, never visible in the Portal.
  - As a **production workflow**: publish the definition draft, create a production workflow against the published version, and drive it via `POST /jobs/{workflow_id}/{action}` and `GET /jobs/{workflow_id}` — the ABAC-scoped resource a real UCP interface would call.
- Three scenarios, run against both test and production workflows: both layers approve; layer 1 rejects; layer 1 cancels. In every case the operator acts directly (they are admin of both the submitting tenant and `clsd-ucp-team` — no impersonation needed).
- A conditional-step scenario, run against production workflows: the network-team step is given a `skip_if` on `params.vlan` and a `complete` trigger whose payload requires `vlan`. Two creations are compared — one with `vlan` in the payload (network-team step skipped) and one without (step waits, then completes with a `vlan` value).
- An approver-group scenario: the first layer uses `user` groups built from `params.approver_emails`, with the resolved approver list compared against the `tenant` kind result. Runs with an empty email list and with `service_resource_role` and no `resource_rns` record how a null group behaves.
- A notification scenario, production workflows only (test workflows send no notifications): an `email` node sends the custom message to addresses passed in `params.approver_emails`. Two variants of the definition are created: one where approving layer 1 leads to `Successful` (control), and one where no path from the approval steps sets `Successful` (ending in `Completed`). The operator records which emails arrive in the inbox for each variant.
- A custom user group kind scenario: tenant secret and config are created, a kind with `oauth2_client_credentials` auth is registered, and its resolution is checked through `users/sync` for more than one tenant. The secret, config and kind are removed afterwards.
- A fingerprint scenario, production workflows only (test workflows send no email): the definition sets `fingerprint_fields` to `resource_type`, `resource_id` and `tenant_name`, and `payload_schema` carries one further field (`reason`) that is not in the fingerprint. Runs compare a definition without a fingerprint (control) against one with it, and cover a duplicate payload while the first workflow is active, the same payload after the first is cancelled, a payload that differs in one fingerprint field, and a payload missing a fingerprint field. The operator records which emails arrive (created, review, updated, rejected), which fields each lists and how they are labeled. Two further runs probe the duplicate check: the same payload created on a newer published version of the Kind while a workflow from the previous version is still active, and several creations of one new payload issued at the same instant. The approver list is the operator only (`user` groups with the operator's email), so no other tenant admin is emailed. A last run measures the width of the race window: pairs of creations of one new payload are issued with the second delayed by 0, 100, 300 and 1000 ms, three pairs per delay, and the number of accepted creations per pair is recorded.
- A fingerprint value-type scenario, production workflows only: `fingerprint_fields` also names fields holding a number, a boolean, an object, an array, a 600-character string and a string with HTML characters. One workflow is created with these values and then repeated to read the `409` body; the operator records how each value is rendered in the review email.
- An idempotency scenario, production workflows only, approver list the operator only. For `X-Request-ID`: a creation with a fresh UUID, a second creation with the same UUID and a different payload, and a third with the same UUID and the first payload, compared with creations that carry no header. For `check-duplicate`: the result for an active payload and for a different payload, then pairs of a creation and a `check-duplicate` call issued 0, 300, 600, 1000 and 1500 ms later, two pairs per delay.
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
6. Against a **production workflow**, exercise the conditional step: add `skip_if`/`skip_with`/`skipped_label` and a `complete` trigger (payload requires `vlan`) to `network_team_approval`, add `vlan` to the definition's `payload_schema`, and create two workflows — one with `vlan` and one without. Record for each: whether the step is skipped or waits, the resulting state history, where the supplied `vlan` is readable afterwards (`params`, or `last_trigger.params` for the manual case), and whether a `data_update` adapter can copy it into `data`.
7. Part B, if ever picked back up: add the callback node back, resolve the reachability question in Risks, and re-run the happy path to observe delivery and retry behavior.

## Success criteria

Test workflows:

- A Workflow Kind and definition draft are created purely through self-service API calls, with no request to the Workflow platform team.
- Layer 1 approver resolution is provably parameter-driven; layer 2 is provably scoped to `clsd-ucp-team` regardless of which tenant submitted the request.
- Reject and cancel at layer 1 both terminate cleanly without layer 2 ever being reached; approving both layers reaches `Successful`.

Production workflows:

- The definition draft publishes successfully and a production workflow can be created against the published version, using the `creator` role.
- The same approver resolution and reject/cancel/approve behavior holds for production workflows as it did for test workflows.
- The production workflow is visible via `GET /jobs` filtered by `created_by` and by `approver` — confirming a UCP-native interface could list and act on it without the OneCloud Portal.

Conditional step:

- A workflow created with `vlan` in its payload passes the network-team step without any human trigger and reaches `Successful` after layer 1 approves.
- A workflow created without `vlan` waits at the network-team step, which only the network team can complete, and the `vlan` supplied at completion is retrievable from the finished workflow.

Approver group:

- The `user` groups resolve to exactly the emails supplied in `params.approver_emails`, and an empty list falls back to the backup group rather than leaving the step unrestricted.

Notifications:

- The custom email arrives at the addresses supplied in `params.approver_emails`, with the request details rendered from `params`.
- For the control variant, the built-in step email arrives alongside the custom one. For the no-`Successful` variant, the operator records whether the built-in step email arrives, and what the end-of-workflow emails to the requester look like.

Custom user group kind:

- The kind registers with `oauth2_client_credentials`, and its resolution through `users/sync` and a real workflow lists exactly the expected approvers, for more than one tenant.

Fingerprint:

- The review email lists the fingerprint fields supplied at creation, and the records state whether the non-fingerprint `reason` field and the other built-in emails show them.
- A second workflow with the same fingerprint values is rejected with `409 WF0011` while the first is active, and the records state whether it is accepted once the first is cancelled and whether a payload that differs in one field is accepted.
- A payload missing a fingerprint field is rejected with `WF0040`.
- The records state whether a creation on a newer published version is refused while a workflow on the previous version is active, and how many of several simultaneous creations of the same payload succeed (one is the expected result), and the largest delay between two creations of one payload at which both are still accepted.

Fingerprint value types:

- The records state how each value type is rendered in the review email (number, boolean, object, array, long string, HTML characters), whether any is truncated or escaped, and how the `409` body reports them.

Idempotency:

- The records state whether a repeated `X-Request-ID` returns the original workflow, creates another one, or is ignored, with and without a changed payload.
- The records state whether `check-duplicate` reports an active payload as a duplicate, and how long after a creation it starts to do so.

Part B (deferred, not being pursued currently):

- The callback reaches a receiver with the expected payload only after both layers approve, and a deliberately-failed delivery is retried by Workflow without manual intervention.

## Risks

- **Callback reachability is unresolved** — every real callback URL found in the research is an internal `r-local.net` service, not a public one, so a laptop script has no obvious way to receive one. This is why Part B is deferred rather than actively blocked-and-waiting: there's no current plan to resolve it, since the approval-chain and visibility questions (Part A) don't depend on it at all.
- **Scope leakage**: it would be easy to drift into building a full Temporal integration, or a real UCP CLI, during this spike. Both remain explicitly out of scope — this PoC only needs to confirm the API surface exists and behaves as expected.
- **Non-representative tenant mapping**: because the PoC's tenant is already a real OneCloud tenant, a successful approver resolution here does not generalize to confirm UCP's actual (possibly independent) tenant model will resolve correctly — that requires the separate conversation with the Workflow team noted in the research doc.

## Open questions

- Same open question as the parent research doc: does UCP's tenant model need a custom user group kind, or does the built-in `tenant` kind suffice? This PoC cannot answer that on its own since it uses an already-OneCloud-native tenant.
- **First-layer approver from a list of emails.** The `service` kind resolves only to `service-provider-admin`, and `service_resource_role` needs a registered Core Data resource plus a fine-grained grant — UCP defines no resources. With an empty `resource_rns` the group resolves to null and is removed, leaving the trigger without approvers. The `user` group list avoids both, at the cost of a fixed slot count and an approver list frozen at creation. Whether the `backup_spec` on a null (as opposed to empty) group takes effect is unverified.
- **Approver notification carrying request details.** The platform sends its own email to the users of a step that can take a required action, automatically and with no opt-out. The documentation describes no way to customize its content. The request's details can be shown on the workflow's Portal detail page through UI customisation (General Parameters card, Additional Parameters cards) and notes supplied at creation. The `email` callback node (`recipient`, `subject`, `body` and `cc`, all Jinja) can send custom content, but approver emails are only exposed to expressions through `last_step.trigger_users`, which exists after a step has been entered, so an email node cannot address a step's approvers at the moment the step opens without resolving them separately first.
- **Fingerprint and a workflow that never ends.** The platform has no step expiry, so a workflow that is never approved or cancelled blocks new requests with the same fingerprint until its requester cancels it. The fingerprint fields must separate requests that legitimately repeat from those that are duplicates.
- Does `GET /jobs` reliably show a workflow to every intended viewer (requester, current-step approvers, viewers), or only some of those roles? Only `created_by` and `approver` filters are exercised here.
