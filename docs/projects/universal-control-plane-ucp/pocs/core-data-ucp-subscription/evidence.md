---
title: "Core Data UCP Subscription Discovery — Evidence"
space: UCP
parent_page_id: "../core-data-ucp-subscription.md"
---

# Evidence: Core Data UCP Subscription Discovery

---

## Horizon Core Data API

| Resource | URL |
|---|---|
| OpenAPI spec (QA) | `https://qa-horizon-data-api.r-local.net/v0/docs/swagger.json` |
| Core Data Overview | [Confluence — AMPORTAL](https://confluence.rakuten-it.com/confluence/display/AMPORTAL/1.+Core+Data+-+Overview) |

### Endpoint used

```
GET /v0/members/{memberRNS}/tenants?subscriptions=true
Authorization: Bearer <keycloak_access_token>
```

**Member RNS format:** `rns:roc:iam:::users:{username}` — must use RNS, not raw email (URL-encoded `@` causes 404).

**Base URLs:**

| Environment | Base URL |
|---|---|
| QA | `https://qa-horizon-data-api.r-local.net/v0` |
| Pre-prod | `https://dev-horizon-data-api.r-local.net/v0` |
| Production | `https://horizon-data-api.r-local.net/v0` |

### Response shape

```json
{
  "total_items": 1,
  "items": [
    {
      "name": "clsd-ucp",
      "rns": "rns:roc:iam::clsd-ucp",
      "subscriptions": [
        { "name": "dbaas", "rns": "rns:roc:dbaas", "title": "Database", "added_at": "..." },
        { "name": "caas",  "rns": "rns:roc:caas",  "title": "Container", "added_at": "..." }
      ]
    }
  ]
}
```

When UCP is subscribed, the expected entry:
```json
{ "name": "ucp", "rns": "rns:roc:ucp", "title": "Universal Control Plane", "added_at": "..." }
```

---

## JWT `groups` claim

Confirmed format from ROC Keycloak QA (`rns:roc:portal` client). Actual test data (raw JWT array, parsed table, Horizon response) is in [poc-report.md](poc-report.md).

### Entry format

```
rns:roc:{service}::{tenant-slug}:roles:{role}
```

| Entry | Meaning |
|---|---|
| `rns:roc:iam::clsd-ucp:roles:admin` | Tenant Admin of clsd-ucp — confirmed present |
| `rns:roc:iam::clsd-ucp:roles:member` | Tenant Member of clsd-ucp — format assumed, unconfirmed for Tenant Member accounts |
| `rns:roc:ucp::clsd-ucp:roles:tenant-admin` | UCP tenant-admin in clsd-ucp — expected format, not yet tested (UCP not yet subscribed in test tenant) |
| `rns:roc:ucp:::roles:service-provider-admin` | UCP service-provider-admin — team-level role, not tenant-scoped; confirmed present in test user's JWT |

When a user has **no role** in a service, that service's entry is **absent** from `groups` — confirmed: DBaaS is subscribed in clsd-ucp but absent from the test user's JWT after their DBaaS role was removed.
