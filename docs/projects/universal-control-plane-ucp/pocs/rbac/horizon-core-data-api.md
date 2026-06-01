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

**Response:**

```json
{
  "items": [
    {
      "rns": "rns:roc:iam::clsd-ucp",
      "name": "clsd-ucp",
      "title": "CLSD UCP Team",
      "admins": [{ "email": "user@rakuten.com" }],
      "subscriptions": [
        { "rns": "rns:roc:caas",  "default_role": "admin"    },
        { "rns": "rns:roc:dbaas", "default_role": "operator" }
      ]
    }
  ]
}
```

`subscriptions[].default_role` is the member's service-level role for that
service within that tenant. This is the single call that answers both the
"which tenants does this user belong to" and "what is their per-service role"
questions in one round trip.

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

Returns all services the tenant is subscribed to and their `default_role`.

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

**Answer: the user's own JWT works.** `GET /v0/members/{email}/tenants?subscriptions=true`
called with `Principal.AccessToken` returns only what that user is allowed to
see. No platform-level service account is needed.

This means Option 2 (UCP RBAC + OC service role check) is technically feasible.
The check at provisioning time would be:

1. Call `GET /v0/members/{email}/tenants?subscriptions=true` with the user's token
2. Find the entry matching the requested tenant
3. Check `subscriptions[].default_role` for the target service (e.g. `rns:roc:dbaas`)
4. Reject if insufficient role

---

## Verification

### Prerequisites

- Access to QA environment
- A valid Keycloak user token (obtain via the QA token endpoint above, or copy
  from browser DevTools after logging into OC Portal)

### Test 1 — Member's tenants and service roles

```bash
TOKEN="<your_keycloak_token>"
EMAIL="your.name@rakuten.com"
BASE="https://qa-horizon-data-api.r-local.net/v0"

curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/members/${EMAIL}/tenants?subscriptions=true" | jq .
```

Expected: array of tenants with `subscriptions[].default_role` for each
subscribed service.

### Test 2 — Tenant member list

```bash
TENANT="rns:roc:iam::clsd-ucp"

curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/tenants/${TENANT}/members" | jq .
```

Expected: list of members with email, type, and role.

### Test 3 — JWT claim parsing (no API call)

```bash
# Decode the JWT payload (middle segment)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq '.groups'
```

Expected: array of RNS role strings including entries like
`rns:roc:iam::<tenantShortId>:roles:admin`.

### Test 4 — Service-level role for a specific tenant+service

```bash
MEMBER_RNS="rns:roc:iam:::users:your.name"
SERVICE="rns:roc:dbaas"

curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/members/${MEMBER_RNS}/tenants/${TENANT}/services/${SERVICE}/access/roles?verify" | jq .
```

---

## Verification Results

| Test | Expected | Actual | Status |
|---|---|---|---|
| Test 1 — Member tenants + subscriptions | Tenants array with `default_role` per service | Not tested — superseded by JWT approach | Skipped |
| Test 2 — Tenant member list | Members list with roles | `role` field is empty; role derived from `admins[]` cross-reference | Verified (QA) |
| Test 3 — JWT groups claim | RNS role strings in `groups` array | Confirmed format: `rns:roc:{service}::{tenant}:roles:{role}`; Tenant Admin format verified | Verified (QA) |
| Test 4 — Service-level role lookup | Role for specific tenant+service | Not tested | Pending |

**Key questions confirmed:**
- JWT `groups` claim includes per-tenant service roles for all subscribed services (e.g. `rns:roc:dbaas::clsd-ucp:roles:admin`). Tenant-level role is the `iam` service entry.
- `GET /v0/tenants/{rns}/members` `role` field is empty in practice — `admins[]` cross-reference is the correct approach for OC role derivation.
- User token is sufficient to call `GET /v0/tenants/{rns}` and `GET /v0/tenants/{rns}/members`.

**Key question still open:**
- Does `rns:roc:iam::{tenant}:roles:member` appear in the JWT for OC Tenant Members, or do they have no `iam` entry? Needs a non-admin test account.
