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
| DBaaS appears in `subscriptions[]` when tenant is subscribed but user has no role | **PASS** |
| DBaaS absent from JWT `groups` when user has no role | **PASS** |
| Horizon API accessible with user's Keycloak access token | **PASS** |
| Subscription status is independent of user role | **PASS — confirmed** |

### Actual JWT `groups` (test run 2026-08-03)

User: `aripermana.putra@rakuten.com` — Tenant Admin of clsd-ucp, DBaaS role removed prior to test.

```json
[
  "rns:roc:lbaas::clsd-ucp:roles:lbaas-viewer",
  "rns:roc:caas::clsd-ucp:roles:admin",
  "rns:roc:ucp:::roles:service-provider-admin",
  "rns:roc:bmaas::clsd-ucp:roles:admin",
  "rns:roc:lbaas::clsd-ucp:roles:lbaas-operator",
  "rns:roc:iam::clsd-ucp:roles:admin",
  "rns:roc:cicd-aas::clsd-ucp:roles:admin",
  "rns:roc:registry-aas::clsd-ucp:roles:admin",
  "rns:roc:computeapi::clsd-ucp:roles:admin",
  "rns:roc:staas::clsd-ucp:roles:admin"
]
```

No `rns:roc:dbaas::*` entry — DBaaS is subscribed in clsd-ucp but the user has no role. This directly proves that subscription status and user role are decoupled in the JWT.

### Actual Horizon response (clsd-ucp subscriptions)

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
        { "name": "bmaas" }
      ]
    }
  ]
}
```

`dbaas` is in `subscriptions[]` ✓ — `ucp` is not yet subscribed in this tenant.

---

## What This PoC Did Not Prove

- UCP subscription appearing in `subscriptions[]` — UCP is not yet subscribed in any test tenant. The DBaaS proxy test confirms the pattern but full end-to-end confirmation with `ucp` in the list requires the ROC team to subscribe a tenant to the UCP service.
- Behavior for Tenant Members (non-admin users) — only tested with a Tenant Admin account.

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

**Open item:** confirm with the ROC team when UCP will be subscribed in the test tenant so the end-to-end flow can be fully verified.
