---
title: "Core Data UCP Subscription Discovery — Design"
space: UCP
parent_page_id: "../core-data-ucp-subscription.md"
---

# Design: Core Data UCP Subscription Discovery

## Research Question

Can the Horizon Core Data API serve as the source of truth for UCP service subscription status, removing the need for UCP to maintain its own `ucp_registered_tenants` table?

**Jira:** MCUCP-130 (ucp tenants list), MCUCP-263

---

## Hypothesis

The Horizon Core Data API endpoint `GET /v0/members/{email}/tenants?subscriptions=true` returns a user's ROC tenant memberships with their service subscription lists. If UCP is registered as a service in Core Data, a subscribed tenant will include `ucp` in its `subscriptions[]` array. Combined with the JWT `groups` claim (which carries the user's UCP role per tenant when assigned), this provides everything needed to render `ucp tenants list` without any server-side UCP state.

---

## Scope

**In scope:**
- Verify that `GET /v0/members/{email}/tenants?subscriptions=true` returns subscription data the CLI can use
- Confirm that a subscribed service appears in `subscriptions[]` regardless of whether the user has a role in it (using DBaaS as a proxy — tenant is subscribed, user has no role)
- Confirm that the JWT `groups` claim is absent for a subscribed service when the user has no role (already confirmed in the keycloak-session-revoke PoC)
- Define the data sources for each field in `ucp tenants list`

**Out of scope:**
- UCP service subscription setup in ROC Portal (UCP is not yet subscribed in the test tenant)
- Testing with a tenant that has UCP subscribed (deferred until ROC team registers UCP service)

---

## Approach

Since UCP is not yet subscribed in any test tenant, DBaaS is used as a functional proxy:

- The `clsd-ucp` tenant is subscribed to DBaaS
- The test user has had their DBaaS role removed
- This mirrors the exact scenario: tenant subscribed to a service, user has no role in it

```mermaid
sequenceDiagram
    participant Script
    participant KC as Keycloak (QA)
    participant HD as Horizon Core Data (QA)

    Script->>KC: PKCE login
    KC-->>Script: access_token (JWT)

    Script->>Script: Parse JWT groups claim
    Note over Script: Check for dbaas entries (expect absent)

    Script->>HD: GET /v0/members/{email}/tenants?subscriptions=true
    HD-->>Script: tenants[] with subscriptions[]
    Note over Script: Check if dbaas is in subscriptions (expect present)

    Script->>Script: Compare JWT groups vs Horizon subscriptions
    Note over Script: dbaas in subscriptions? YES<br/>dbaas in JWT groups? NO → confirms decoupling
```

**Success criteria:**

| # | Criterion | Expected |
|---|---|---|
| SC-1 | DBaaS appears in `subscriptions[]` for clsd-ucp tenant | Pass |
| SC-2 | JWT `groups` claim has no DBaaS entry (user has no role) | Pass |
| SC-3 | `iam` entry for clsd-ucp is present in JWT (tenant membership) | Pass |
| SC-4 | Horizon API accessible with user's access token directly | Pass |

---

## Proposed data flow for `ucp tenants list`

```mermaid
flowchart TD
    A[CLI: ucp tenants list] --> B[Decode stored JWT locally]
    B --> C[Parse rns:roc:iam:: entries\n→ user's ROC tenants + OC role]
    C --> D[Call GET /api/v1/me/tenants on API server]
    D --> E[API server: call Horizon\nGET /v0/members/email/tenants?subscriptions=true]
    E --> F[Filter tenants where ucp in subscriptions]
    F --> G[For each UCP-subscribed tenant:\nparse rns:roc:ucp::tenant:roles:role from JWT\n→ user's UCP role, null if absent]
    G --> H[Return filtered list to CLI]
    H --> I[CLI renders table]
```

---

## Risks and Open Questions

1. **UCP not yet subscribed in any test tenant** — DBaaS is used as a proxy. Full end-to-end confirmation with UCP requires the ROC team to subscribe a tenant to the UCP service.
2. **Horizon API network accessibility** — `qa-horizon-data-api.r-local.net` requires access to the Rakuten internal network. Confirm reachability from the kitchen-sink script.
3. **Latency** — calling Horizon at every `ucp tenants list` invocation adds one API call. Acceptable for a list command; not acceptable for every protected API endpoint.
