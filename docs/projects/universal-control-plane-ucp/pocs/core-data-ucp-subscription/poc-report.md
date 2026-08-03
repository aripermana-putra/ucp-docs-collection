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
