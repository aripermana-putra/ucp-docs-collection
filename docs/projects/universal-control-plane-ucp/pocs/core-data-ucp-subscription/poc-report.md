---
title: "Core Data UCP Subscription Discovery — PoC Report"
space: UCP
parent_page_id: "../core-data-ucp-subscription.md"
---

# PoC Report: Core Data UCP Subscription Discovery

**Research question answered:** Can the Horizon Core Data API serve as the source of truth for UCP service subscription status, removing the need for a UCP-maintained `ucp_registered_tenants` table?

**Status:** Complete

---

## Verdict

**Yes — Horizon Core Data is the correct source of truth for UCP subscription status.** A single call to `GET /v0/members/{rns}/tenants?subscriptions=true` returns all tenants the user belongs to with their subscription lists. A tenant is UCP-subscribed when `ucp` appears in its `subscriptions[]` array. This is independent of whether the user has a UCP role in that tenant.

The `ucp_registered_tenants` DB table is not needed.

---

## What This PoC Proved

| Criterion | Result |
|---|---|
| DBaaS appears in `subscriptions[]` when tenant is subscribed but user has no role | **PASS** (2026-08-03) |
| DBaaS absent from JWT `groups` when user has no role | **PASS** (2026-08-03) |
| UCP appears in `subscriptions[]` when tenant subscribes to UCP | **PASS** (2026-08-04) |
| No tenant-scoped UCP role in JWT when no role assigned | **PASS** (2026-08-04) |
| Multiple UCP roles for same tenant all appear as separate JWT entries | **PASS** (2026-08-04) |
| Horizon API accessible with user's Keycloak access token | **PASS** |
| Subscription status is independent of user role | **PASS — confirmed** |

### Actual JWT `groups` (test run 2026-08-04)

User: `aripermana.putra@rakuten.com` — Tenant Admin of clsd-ucp, no UCP tenant role assigned.

```json
[
  "rns:roc:lbaas::clsd-ucp:roles:lbaas-operator",
  "rns:roc:cicd-aas::clsd-ucp:roles:admin",
  "rns:roc:registry-aas::clsd-ucp:roles:admin",
  "rns:roc:lbaas::clsd-ucp:roles:lbaas-viewer",
  "rns:roc:caas::clsd-ucp:roles:admin",
  "rns:roc:computeapi::clsd-ucp:roles:admin",
  "rns:roc:ucp:::roles:service-provider-admin",
  "rns:roc:bmaas::clsd-ucp:roles:admin",
  "rns:roc:iam::clsd-ucp:roles:admin",
  "rns:roc:staas::clsd-ucp:roles:admin"
]
```

No `rns:roc:ucp::clsd-ucp:roles:*` entry — clsd-ucp is subscribed to UCP but the user has no tenant-scoped UCP role assigned. `rns:roc:ucp:::roles:service-provider-admin` is the team-level UCP service role (empty tenant slug) — unrelated to tenant user roles.

### JWT `groups` with all 3 UCP roles assigned (test run 2026-08-04)

After assigning `viewer`, `developer`, and `tenant-admin` to the same user in OneCloud:

```json
[
  "rns:roc:ucp:::roles:service-provider-admin",
  "rns:roc:ucp::clsd-ucp:roles:viewer",
  "rns:roc:ucp::clsd-ucp:roles:developer",
  "rns:roc:ucp::clsd-ucp:roles:tenant-admin",
  "rns:roc:iam::clsd-ucp:roles:admin",
  ... (other services)
]
```

All three tenant-scoped roles appear as **separate entries** in the JWT `groups` claim. OneCloud allows multiple roles per service per tenant. The `RequirePermission` middleware must scan **all** matching `rns:roc:ucp::clsd-ucp:roles:*` entries and OR their permission sets together to get the effective permission for the request.

### Actual Horizon response (clsd-ucp subscriptions, 2026-08-04)

```json
{
  "total_items": 1,
  "items": [
    {
      "name": "clsd-ucp",
      "rns": "rns:roc:iam::clsd-ucp",
      "subscriptions": [
        { "name": "computeapi" },
        { "name": "caas" },
        { "name": "lbaas" },
        { "name": "staas" },
        { "name": "cicd-aas" },
        { "name": "billing" },
        { "name": "dbaas" },
        { "name": "registry-aas" },
        { "name": "bmaas" },
        { "name": "ucp" }
      ]
    }
  ]
}
```

`ucp` is now in `subscriptions[]` ✓ — full end-to-end scenario confirmed: tenant subscribed to UCP, user has no tenant-scoped UCP role → `ucpRole = null`.

---

## What This PoC Did Not Prove

- Behavior for Tenant Members (non-admin users) — only tested with a Tenant Admin account. Whether a Tenant Member (not admin) has a `rns:roc:iam::{tenant}:roles:member` entry in their JWT `groups` claim remains unconfirmed.

---

## Recommendation

**Use `GET /v0/members/{rns}/tenants?subscriptions=true` as the data source for `ucp tenants list`.**

Data model for the response:

| Field | Source |
|---|---|
| Tenant list | JWT `rns:roc:iam::` entries |
| OC role | JWT `rns:roc:iam::{tenant}:roles:{role}` |
| UCP subscription | Horizon `subscriptions[].name == "ucp"` |
| UCP role | JWT `rns:roc:ucp::{tenant}:roles:{role}` (null if absent) |

This is one Horizon call per `ucp tenants list` invocation — acceptable for a list command. Not suitable for per-request authorization (MCUCP-138 uses JWT-only for that).

**Open item:** confirm behavior for Tenant Member accounts — whether `rns:roc:iam::{tenant}:roles:member` appears in their JWT `groups` claim.
