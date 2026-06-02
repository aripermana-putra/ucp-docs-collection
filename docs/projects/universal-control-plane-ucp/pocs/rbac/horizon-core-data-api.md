---
title: "RBAC — Horizon Core Data API"
space: UCP
parent_page_id: "../rbac.md"
---

# Horizon Core Data API — Member & Tenant Identity

Research into the Core Data API endpoints needed to fetch member data (which
tenants a user belongs to and their roles) and tenant data (who are the members
of a tenant and what roles they hold). Relevant to the PM's Option 2 feasibility
question.

---

## References

| Resource | Link |
|---|---|
| OpenAPI spec (QA) | [swagger.json](https://qa-horizon-data-api.r-local.net/v0/docs/swagger.json) |
| Core Data — Overview | [Confluence — AMPORTAL](https://confluence.rakuten-it.com/confluence/display/AMPORTAL/1.+Core+Data+-+Overview) |
| Core Data — API registry | [Confluence — [Review] Horizon Core Data](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6125028565) |
| Core Data — Release changelog | [Confluence — Core Data releases](https://confluence.rakuten-it.com/confluence/display/CLDCPS/Core+Data+releases) |
| PM requirements (Draft) | [UCP Identity, Tenancy & Roles](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6645566515/UCP+Identity+Tenancy+Roles) |

---

## Base URLs

| Environment | Base URL |
|---|---|
| QA | `https://qa-horizon-data-api.r-local.net/v0` |
| Pre-prod | `https://dev-horizon-data-api.r-local.net/v0` |
| Production | `https://horizon-data-api.r-local.net/v0` |

---

## Authentication

Every call requires a Keycloak OAuth2 Bearer token:

```
Authorization: Bearer <token>
```

### Token types

| Type | grant_type | Use case |
|---|---|---|
| User token | `password` | Human user calls — carries that user's tenant roles |
| Service account token | `client_credentials` | Machine-to-machine calls |

### Keycloak token endpoints

| Environment | Endpoint |
|---|---|
| QA | `https://qa2-accounts-onecloud.rakuten-it.com/auth/realms/roc/protocol/openid-connect/token` |
| Pre-prod | `https://dev-accounts-onecloud.rakuten-it.com/auth/realms/roc/protocol/openid-connect/token` |
| Production | `https://accounts-onecloud.rakuten-it.com/auth/realms/roc/protocol/openid-connect/token` |

UCP already has the user's access token decrypted and available as
`Principal.AccessToken` (injected by `SessionMiddleware`). No new credential
setup is needed to call these endpoints on behalf of the current user.

---

## Use Case 1 — Member data

### Tenants a member belongs to (with service roles)

```
GET /v0/members/{memberIdentifier}/tenants?subscriptions=true
Authorization: Bearer {user_token}
```

`memberIdentifier` is the user's email or RNS (`rns:roc:iam:::users:{username}`).

**Actual response (QA, clsd-ucp tenant):**

```json
{
  "total_items": 1,
  "items": [
    {
      "name": "clsd-ucp",
      "rns": "rns:roc:iam::clsd-ucp",
      "title": "clsd-ucp",
      "service_id": "100772",
      "billing_identifier": "100772",
      "subscriptions": [
        { "name": "computeapi", "rns": "rns:roc:computeapi", "title": "Virtual Machine",         "added_at": "2026-02-26T09:01:51Z" },
        { "name": "caas",       "rns": "rns:roc:caas",       "title": "Container",               "added_at": "2026-02-19T06:33:23Z" },
        { "name": "lbaas",      "rns": "rns:roc:lbaas",      "title": "Load Balancer",           "added_at": "2026-05-11T07:29:28Z" },
        { "name": "staas",      "rns": "rns:roc:staas",      "title": "Storage",                 "added_at": "2026-02-26T08:36:35Z" },
        { "name": "cicd-aas",   "rns": "rns:roc:cicd-aas",   "title": "CICD-as-a-Service",       "added_at": "2026-02-26T08:30:55Z" },
        { "name": "billing",    "rns": "rns:roc:billing",    "title": "ROC Billing",             "default_role": "admin", "added_at": "2026-01-20T08:18:50Z" },
        { "name": "dbaas",      "rns": "rns:roc:dbaas",      "title": "Database",                "added_at": "2026-01-21T01:39:16Z" },
        { "name": "registry-aas","rns": "rns:roc:registry-aas","title": "Registry as a Service", "added_at": "2026-02-19T06:33:23Z" },
        { "name": "bmaas",      "rns": "rns:roc:bmaas",      "title": "Baremetal",               "added_at": "2026-02-26T08:26:26Z" }
      ]
    }
  ]
}
```

`subscriptions[].default_role` is **not reliably present** — only `billing`
returned one in testing. Most services omit the field. Per-member service roles
should be read from the JWT `groups` claim or from
`GET /v0/members/{memberRNS}/tenants/{tenantRNS}/services/{serviceRNS}/access/roles`
(Test 4) instead. This endpoint is also missing the `admins[]` field documented
in the Horizon API reference — it was not present in the actual response.

> **Note:** In the current implementation this endpoint is **not called**. The
> logged-in user's own tenant list and OC tenant role are derived entirely from
> the Keycloak JWT `groups` claim via `parseOCGroupsFromJWT` — zero Horizon
> calls for the logged-in user's own data. This endpoint remains documented for
> reference and for Option 2 feasibility analysis.

### Service-level role for a specific member/tenant/service

```
GET /v0/members/{memberRNS}/tenants/{tenantRNS}/services/{serviceRNS}/access/roles
Authorization: Bearer {token}
```

Append `?verify` to bypass cache and force fresh data:

```
GET /v0/members/{memberRNS}/tenants/{tenantRNS}/services/{serviceRNS}/access/roles?verify
```

---

## Use Case 2 — Tenant data

### List all members of a tenant

```
GET /v0/tenants/{tenantRNS}/members
Authorization: Bearer {token}
```

Returns each member's profile (email, name, type). The `role` field in the
response is empty in practice — tenant-level role (`Tenant Admin` or
`Tenant Member`) must be derived by cross-referencing the `admins[]` array
from `GET /v0/tenants/{tenantRNS}`: email present in `admins[]` → "Tenant Admin",
otherwise → "Tenant Member".

### Tenant details including admins

```
GET /v0/tenants/{tenantRNS}
Authorization: Bearer {token}
```

**Response:**

```json
{
  "rns": "rns:roc:iam::clsd-ucp",
  "name": "clsd-ucp",
  "title": "CLSD UCP Team",
  "admins": [
    { "email": "user@rakuten.com", "name": "User Name", "type": "user" }
  ]
}
```

The `admins[]` array is the authoritative source for determining which members
hold the Tenant Admin role, since `GET /v0/tenants/{rns}/members` does not
return a reliable role field.

### Tenant subscriptions

```
GET /v0/tenants/{tenantRNS}/subscriptions
Authorization: Bearer {token}
```

Returns all services the tenant is subscribed to. `default_role` is absent for
most services — confirmed by testing (see Verification Results).

**Actual response (QA, clsd-ucp tenant):**

```json
{
  "total_items": 9,
  "items": [
    { "name": "computeapi",  "rns": "rns:roc:computeapi",  "title": "Virtual Machine",         "added_at": "2026-02-26T09:01:51Z" },
    { "name": "caas",        "rns": "rns:roc:caas",        "title": "Container",               "added_at": "2026-02-19T06:33:23Z" },
    { "name": "lbaas",       "rns": "rns:roc:lbaas",       "title": "Load Balancer",           "added_at": "2026-05-11T07:29:28Z" },
    { "name": "staas",       "rns": "rns:roc:staas",       "title": "Storage",                 "added_at": "2026-02-26T08:36:35Z" },
    { "name": "cicd-aas",    "rns": "rns:roc:cicd-aas",    "title": "CICD-as-a-Service",       "added_at": "2026-02-26T08:30:55Z" },
    { "name": "billing",     "rns": "rns:roc:billing",     "title": "ROC Billing", "default_role": "admin", "added_at": "2026-01-20T08:18:50Z" },
    { "name": "dbaas",       "rns": "rns:roc:dbaas",       "title": "Database",                "added_at": "2026-01-21T01:39:16Z" },
    { "name": "registry-aas","rns": "rns:roc:registry-aas","title": "Registry as a Service",   "added_at": "2026-02-19T06:33:23Z" },
    { "name": "bmaas",       "rns": "rns:roc:bmaas",       "title": "Baremetal",               "added_at": "2026-02-26T08:26:06Z" }
  ]
}
```

---

## Alternative: Read roles directly from the Keycloak JWT

The Keycloak JWT `groups` claim already carries all role assignments as RNS
strings — no API call needed:

```json
{
  "groups": [
    "rns:roc:iam::a7e1393a:roles:admin",
    "rns:roc:caas:::roles:service-provider-admin",
    "rns:roc:dbaas:::roles:service-provider-admin"
  ]
}
```

**Role RNS format:** `rns:roc:<service>::<tenant>[resource]:roles:<role>`

UCP already has the access token as `Principal.AccessToken`. Parsing the
`groups` claim from this JWT returns all roles with zero additional network
calls.

**Trade-off:**

| | JWT claim approach | Core Data API approach |
|---|---|---|
| Latency | 0 extra calls | 1 API call |
| Freshness | Stale by up to token TTL (~10 min) | Live data |
| Implementation | Parse JWT payload | Call `/v0/members/{email}/tenants?subscriptions=true` |

---

## Relevance to PM's Option 2

The PM's open question was: *"Can UCP call Core Data API using the user's own
JWT, or does UCP need a dedicated platform-level service account?"*

**Answer: the user's own JWT works.** All tested endpoints (`GET /v0/members`,
`GET /v0/tenants`) return data scoped to what that user is allowed to see when
called with `Principal.AccessToken`. No platform-level service account is needed.

### Which endpoint to use for Option 2

Testing ruled out `subscriptions[].default_role` (Tests 1 and 5) — it is absent
for most services. The only reliable real-time service role check is:

```
GET /v0/members/{memberRNS}/tenants/{tenantRNS}/services/{serviceRNS}/access/roles?verify
```

The `?verify` flag bypasses Horizon's cache and forces a live lookup. Check
`items[0].name` against the required role for the resource type being provisioned
(e.g. `"admin"` or `"operator"` for DBaaS).

### Two implementation options for Option 2

| Approach | Freshness | Horizon call at provisioning | Implementation |
|---|---|---|---|
| **Live check** — call `access/roles?verify` at provisioning time | Real-time | 1 call per provision request | Wire `RequirePermission` middleware to call Horizon |
| **Cached check** — read `oc_roles.oc_service_roles` (JWT-populated on login) | Stale up to token TTL (~10 min) | 0 calls | Wire middleware to query local DB |

The cached approach uses data already collected — `oc_roles.oc_service_roles`
is populated on every login from the JWT `groups` claim with no additional
Horizon calls. Enabling it requires only a middleware wire-up, no schema changes.

The live approach is the only option when real-time accuracy is required (e.g.
a user's OC service role was downgraded since their last login).

---

## Verification

Tests were run via the UCP API server proxy endpoints against QA environment
(`X-Environment: QA`), using the session token from an authenticated UCP session.
Underlying Horizon base URL: `https://qa-horizon-data-api.r-local.net/v0`.

---

### Test 1 — Member's tenants and subscriptions

Horizon endpoint: `GET /v0/members/{email}/tenants?subscriptions=true`

```bash
curl -s -H "Cookie: session=<session>" \
  -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/horizon/tenants/member?member=aripermana.putra@rakuten.com" | jq .
```

**Response:**

```json
{
  "total_items": 1,
  "items": [
    {
      "name": "clsd-ucp",
      "rns": "rns:roc:iam::clsd-ucp",
      "title": "clsd-ucp",
      "service_id": "100772",
      "billing_identifier": "100772",
      "subscriptions": [
        { "name": "computeapi",  "rns": "rns:roc:computeapi",  "title": "Virtual Machine",          "added_at": "2026-02-26T09:01:51Z" },
        { "name": "caas",        "rns": "rns:roc:caas",        "title": "Container",                "added_at": "2026-02-19T06:33:23Z" },
        { "name": "lbaas",       "rns": "rns:roc:lbaas",       "title": "Load Balancer",            "added_at": "2026-05-11T07:29:28Z" },
        { "name": "staas",       "rns": "rns:roc:staas",       "title": "Storage",                  "added_at": "2026-02-26T08:36:35Z" },
        { "name": "cicd-aas",    "rns": "rns:roc:cicd-aas",    "title": "CICD-as-a-Service",        "added_at": "2026-02-26T08:30:55Z" },
        { "name": "billing",     "rns": "rns:roc:billing",     "title": "ROC Billing",     "default_role": "admin", "added_at": "2026-01-20T08:18:50Z" },
        { "name": "dbaas",       "rns": "rns:roc:dbaas",       "title": "Database",                 "added_at": "2026-01-21T01:39:16Z" },
        { "name": "registry-aas","rns": "rns:roc:registry-aas","title": "Registry as a Service",    "added_at": "2026-02-19T06:33:23Z" },
        { "name": "bmaas",       "rns": "rns:roc:bmaas",       "title": "Baremetal",                "added_at": "2026-02-26T08:26:26Z" }
      ]
    }
  ]
}
```

**Finding:** `default_role` is absent for most services — only `billing` returned
one. This endpoint is **not a reliable source for per-member service roles**.
Use the JWT `groups` claim or the service role endpoint (Test 4) instead.

---

### Test 2 — Tenant member list

Horizon endpoint: `GET /v0/tenants/{tenantRNS}/members`

```bash
curl -s -H "Cookie: session=<session>" \
  -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/horizon/tenants/rns%3Aroc%3Aiam%3A%3Aclsd-ucp/members" | jq .
```

**Response:**

```json
{
  "total_items": 7,
  "items": [
    {
      "added_at": "2026-05-28T01:22:58Z",
      "email": "sebasti.bellefeuille@rakuten.com",
      "firstname": "Sebastien",
      "last_login": "2026-05-28T14:09:54+09:00",
      "lastname": "Bellefeuille",
      "name": "sebasti.bellefeuille",
      "rns": "rns:roc:iam:::users:sebasti.bellefeuille",
      "title": "Sebastien Bellefeuille",
      "type": "user",
      "username": "sebasti.bellefeuille"
    },
    {
      "added_at": "2026-01-20T08:18:50Z",
      "email": "ryo.kimura@rakuten.com",
      "firstname": "Ryo",
      "last_login": "2026-05-22T10:22:51+09:00",
      "lastname": "Kimura",
      "name": "ryo.kimura",
      "rns": "rns:roc:iam:::users:ryo.kimura",
      "title": "Ryo Kimura",
      "type": "user",
      "username": "ryo.kimura"
    },
    {
      "added_at": "2026-01-21T01:39:16Z",
      "email": "yusuke.a.ohashi@rakuten.com",
      "firstname": "Yusuke a",
      "last_login": "2026-05-29T14:50:38+09:00",
      "lastname": "Ohashi",
      "name": "yusuke.a.ohashi",
      "rns": "rns:roc:iam:::users:yusuke.a.ohashi",
      "title": "Yusuke a Ohashi",
      "type": "user",
      "username": "yusuke.a.ohashi"
    },
    {
      "added_at": "2026-05-28T05:10:59Z",
      "email": "rania.benkahla@rakuten.com",
      "firstname": "Rania",
      "last_login": "2026-05-29T10:59:35+09:00",
      "lastname": "Ben Kahla",
      "name": "rania.benkahla",
      "rns": "rns:roc:iam:::users:rania.benkahla",
      "title": "Rania Ben Kahla",
      "type": "user",
      "username": "rania.benkahla"
    },
    {
      "added_at": "2026-04-07T02:53:05Z",
      "email": "aripermana.putra@rakuten.com",
      "firstname": "Ari Permana",
      "last_login": "2026-05-29T15:50:57+09:00",
      "lastname": "Putra",
      "name": "aripermana.putra",
      "rns": "rns:roc:iam:::users:aripermana.putra",
      "title": "Ari Permana Putra",
      "type": "user",
      "username": "aripermana.putra"
    },
    {
      "added_at": "2026-05-01T05:54:52Z",
      "email": "aya.wang@rakuten.com",
      "firstname": "Aya",
      "last_login": "2026-05-26T15:04:07+09:00",
      "lastname": "Wang",
      "name": "aya.wang",
      "rns": "rns:roc:iam:::users:aya.wang",
      "title": "Aya Wang",
      "type": "user",
      "username": "aya.wang"
    },
    {
      "added_at": "2026-05-11T03:08:06Z",
      "email": "ucp-cli-auth@clsd-ucp.iam.service-accounts.roc",
      "name": "ucp-cli-auth",
      "rns": "rns:roc:iam::clsd-ucp:service-accounts:ucp-cli-auth",
      "title": "ucp-cli-auth",
      "type": "service-account"
    }
  ]
}
```

**Finding:** No `role` field anywhere in the response — confirmed. Tenant-level
role must be derived from the `admins[]` array on `GET /v0/tenants/{rns}`.
The response also includes a `type: "service-account"` member, confirming
that non-user member types exist in the member list (see open question on
non-user member handling).

---

### Test 3 — JWT groups claim

No Horizon call. Parses `groups` from the session's access token.

```bash
curl -s -H "Cookie: session=<session>" \
  "http://localhost:8080/api/v1/horizon/jwt/groups" | jq .
```

**Response:**

```json
{
  "rawGroups": [
    "rns:roc:caas::clsd-ucp:roles:admin",
    "rns:roc:lbaas::clsd-ucp:roles:lbaas-operator",
    "rns:roc:staas::clsd-ucp:roles:admin",
    "rns:roc:lbaas::clsd-ucp:roles:lbaas-viewer",
    "rns:roc:iam::clsd-ucp:roles:admin",
    "rns:roc:registry-aas::clsd-ucp:roles:admin",
    "rns:roc:dbaas::clsd-ucp:roles:admin",
    "rns:roc:bmaas::clsd-ucp:roles:admin",
    "rns:roc:cicd-aas::clsd-ucp:roles:admin",
    "rns:roc:computeapi::clsd-ucp:roles:admin"
  ],
  "parsed": [
    {
      "RNS": "rns:roc:iam::clsd-ucp",
      "Name": "",
      "Role": "Tenant Admin",
      "ServiceRoles": {
        "bmaas": "admin",
        "caas": "admin",
        "cicd-aas": "admin",
        "computeapi": "admin",
        "dbaas": "admin",
        "lbaas": "lbaas-viewer",
        "registry-aas": "admin",
        "staas": "admin"
      }
    }
  ]
}
```

**Finding:** JWT `groups` format confirmed as `rns:roc:{service}::{tenant-slug}:roles:{role}`.
The `iam` entry (`rns:roc:iam::clsd-ucp:roles:admin`) is the tenant-level role.
A user can have multiple entries for the same service with different roles
(`lbaas-operator` and `lbaas-viewer` both present) — `parseOCGroupsFromJWT`
takes the last one seen; consider using the higher-privilege role if multiples
exist. The JWT encodes all service roles in a single token with zero API calls.

---

### Test 4 — Service-level role for a specific tenant+service

Horizon endpoint: `GET /v0/members/{memberRNS}/tenants/{tenantRNS}/services/{serviceRNS}/access/roles?verify`

```bash
curl -s -H "Cookie: session=<session>" \
  -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/horizon/member/service-role\
?memberRNS=rns:roc:iam:::users:aripermana.putra\
&tenantRNS=rns:roc:iam::clsd-ucp\
&serviceRNS=rns:roc:dbaas" | jq .
```

**Response:**

```json
{
  "total_items": 1,
  "items": [
    {
      "name": "admin",
      "added_at": "2026-04-07T02:53:05Z",
      "rns": "rns:roc:dbaas:::roles:admin"
    }
  ]
}
```

**Finding:** Endpoint works. Returns the role as a named item with its RNS.
User token is sufficient — no service account required. This is the reliable
per-member service role check for Option 2, since `subscriptions[].default_role`
(Test 1) is not reliably populated.

---

### Test 5 — Tenant subscriptions

Horizon endpoint: `GET /v0/tenants/{tenantRNS}/subscriptions`

```bash
curl -s -H "Cookie: session=<session>" \
  -H "X-Environment: QA" \
  "http://localhost:8080/api/v1/horizon/tenants/rns%3Aroc%3Aiam%3A%3Aclsd-ucp/subscriptions" | jq .
```

**Response:**

```json
{
  "total_items": 9,
  "items": [
    { "name": "computeapi",  "rns": "rns:roc:computeapi",  "title": "Virtual Machine",         "added_at": "2026-02-26T09:01:51Z" },
    { "name": "caas",        "rns": "rns:roc:caas",        "title": "Container",               "added_at": "2026-02-19T06:33:23Z" },
    { "name": "lbaas",       "rns": "rns:roc:lbaas",       "title": "Load Balancer",           "added_at": "2026-05-11T07:29:28Z" },
    { "name": "staas",       "rns": "rns:roc:staas",       "title": "Storage",                 "added_at": "2026-02-26T08:36:35Z" },
    { "name": "cicd-aas",    "rns": "rns:roc:cicd-aas",    "title": "CICD-as-a-Service",       "added_at": "2026-02-26T08:30:55Z" },
    { "name": "billing",     "rns": "rns:roc:billing",     "title": "ROC Billing", "default_role": "admin", "added_at": "2026-01-20T08:18:50Z" },
    { "name": "dbaas",       "rns": "rns:roc:dbaas",       "title": "Database",                "added_at": "2026-01-21T01:39:16Z" },
    { "name": "registry-aas","rns": "rns:roc:registry-aas","title": "Registry as a Service",   "added_at": "2026-02-19T06:33:23Z" },
    { "name": "bmaas",       "rns": "rns:roc:bmaas",       "title": "Baremetal",               "added_at": "2026-02-26T08:26:06Z" }
  ]
}
```

**Finding:** Response shape is identical to the `subscriptions[]` array in Test 1.
`default_role` is absent for all services except `billing`, consistent with
Test 1. This endpoint tells you what services the tenant is subscribed to but
does not carry per-member role information — it is not useful for Option 2
role checks.

---

## Verification Results

| Test | Horizon endpoint | Status | Key finding |
|---|---|---|---|
| Test 1 — Member tenants + subscriptions | `GET /v0/members/{email}/tenants?subscriptions=true` | Verified (QA) | `default_role` absent for most services — not reliable for role checks |
| Test 2 — Tenant member list | `GET /v0/tenants/{rns}/members` | Verified (QA) | No `role` field; includes `service-account` type members |
| Test 3 — JWT groups claim | n/a — JWT parse only | Verified (QA) | Format confirmed; multiple roles per service possible |
| Test 4 — Service-level role lookup | `GET /v0/members/{rns}/tenants/{rns}/services/{rns}/access/roles?verify` | Verified (QA) | Works with user token; returns role name + RNS |
| Test 5 — Tenant subscriptions | `GET /v0/tenants/{rns}/subscriptions` | Verified (QA) | Same shape as Test 1 `subscriptions[]`; `default_role` absent except `billing` |

**Confirmed:**
- JWT `groups` claim encodes all service roles per tenant — the correct source for per-member service role data at login time.
- `GET /v0/tenants/{rns}/members` has no `role` field — `admins[]` cross-reference from `GET /v0/tenants/{rns}` is required.
- `subscriptions[].default_role` is absent for most services — do not use for role checks.
- `GET /v0/members/{rns}/tenants/{rns}/services/{rns}/access/roles?verify` is the correct live service role check for Option 2 at provisioning time.
- User token is sufficient for all tested endpoints — no platform service account needed.

**Still open:**
- Does `rns:roc:iam::{tenant}:roles:member` appear in the JWT for OC Tenant Members, or do they have no `iam` entry? Needs a non-admin test account.
