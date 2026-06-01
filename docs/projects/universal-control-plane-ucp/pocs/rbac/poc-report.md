---
title: "POC Report — RBAC + Tenant Onboarding"
space: UCP
parent_page_id: "../rbac.md"
---

# Role-Based Access Control (RBAC) + Tenant Onboarding

| | |
|---|---|
| **Jira** | [MCUCP-191](https://jira.rakuten-it.com/jira/browse/MCUCP-191) |
| **Author** | aripermana.putra |
| **Date** | 2026-06-01 |
| **Status** | PROPOSED |

---

## 1. Summary

MCUCP-191 proves that UCP can enforce per-tenant role-based access control and
automate tenant onboarding from OC Core Data. A 3-role permission model
(`developer`, `tenant-admin`, `platform-admin`) using a bitmask design is
deployed across all API endpoints and the frontend. Tenant membership and OC
roles are synchronised from Horizon on every login — storing all members locally
so that role management requires zero live Horizon calls at request time.

**Verdict: Pending.** The permission model, onboarding flow, and member sync all
work end-to-end. Several open questions remain (see Section 4).

---

## 2. Objectives & Success Criteria

**Hypothesis:**
UCP can enforce per-tenant RBAC without duplicating OC's access model — by
reading OC membership from Keycloak JWTs and Horizon Core Data once per login
and enforcing locally stored roles on every subsequent request.

**Success criteria:**

| # | Criterion | Result |
|---|---|---|
| SC-1 | A `developer` calling `GET /api/v1/databases` receives 200 | Pass |
| SC-2 | A `developer` calling `POST /api/v1/databases` succeeds (202) | Pass |
| SC-3 | A `developer` calling `POST .../approve` receives 403 | Pass |
| SC-4 | A `developer` calling `GET /settings/credentials` receives 403 | Pass |
| SC-5 | A `tenant-admin` calling `POST /settings/credentials` succeeds | Pass |
| SC-6 | A user with no UCP role sees only the Tenants menu; all resource menus hidden | Pass |
| SC-7 | OC Tenant Admin can register their tenant without a pre-existing UCP role | Pass |
| SC-8 | All OC Tenant Admins with UCP accounts receive `tenant-admin` on registration | Pass |
| SC-9 | All OC tenant members appear in the member picker after login sync | Pass |

**Scope boundaries (out of scope):**
- Keycloak configuration changes
- OC service-level role enforcement at provisioning time (Option 2)
- Pre-assigning UCP roles to members who have never logged in
- Periodic background sync between logins

---

## 3. Findings

### Permission model

A **permission bitmask** (`PermRead | PermProvision | PermApprove | PermManage | PermPlatform`)
correctly separates `developer` and `tenant-admin` — a `developer` cannot approve
their own provisioning request since `PermApprove` is absent from their bitmask.
Adding a new role in future requires one line in the `RolePermissions` map with
no schema changes.

### Login sync

The Keycloak JWT `groups` claim encodes all of the logged-in user's OC tenant
memberships and per-service roles. The logged-in user's own tenant list and OC
role are derived entirely from the JWT — no Horizon call needed for their own data.

**Actual JWT `groups` claim (QA, Tenant Admin account):**

```json
{
  "iss": "https://qa2-accounts-onecloud.rakuten-it.com/auth/realms/roc",
  "sub": "<keycloak-user-id>",
  "email": "user@rakuten.com",
  "preferred_username": "user.name",
  "groups": [
    "rns:roc:iam::clsd-ucp:roles:admin",
    "rns:roc:dbaas::clsd-ucp:roles:admin",
    "rns:roc:caas::clsd-ucp:roles:admin",
    "rns:roc:staas::clsd-ucp:roles:admin",
    "rns:roc:lbaas::clsd-ucp:roles:lbaas-operator",
    "rns:roc:lbaas::clsd-ucp:roles:lbaas-viewer",
    "rns:roc:bmaas::clsd-ucp:roles:admin",
    "rns:roc:cicd-aas::clsd-ucp:roles:admin",
    "rns:roc:registry-aas::clsd-ucp:roles:admin",
    "rns:roc:computeapi::clsd-ucp:roles:admin"
  ]
}
```

**How UCP reads it:**

Each entry follows the format `rns:roc:{service}::{tenant-slug}:roles:{role}`.
On every login, UCP parses the `groups` array and derives:

- **Tenant membership** — which tenants the user belongs to (from the `iam` service entry)
- **OC tenant role** — `roles:admin` → "Tenant Admin", `roles:member` → "Tenant Member"
- **OC service roles** — all non-`iam` entries per tenant, stored as a JSONB map
  (e.g. `{"dbaas": "admin", "caas": "admin", "lbaas": "lbaas-viewer"}`)

The parsed data is written to the local DB (`oc_roles`) and drives all subsequent
UCP behaviour for that session — no further Horizon calls at request time.

Note: a user can have multiple entries for the same service (e.g. `lbaas-operator`
and `lbaas-viewer` both present above). UCP takes the last one parsed; resolving
the correct role when multiple exist is an edge case to address before MVP.

The `groups` claim grows linearly with the number of tenant memberships and
service subscriptions — users with many tenants will have proportionally larger
tokens. This is a production sizing concern to account for in MVP capacity planning.

OC role for **other** tenant members cannot be read from `GET /v0/tenants/{rns}/members`
— the `role` field is empty in the actual Horizon response. The correct approach
is to cross-reference the `admins[]` array from `GET /v0/tenants/{rns}`:
email present in `admins[]` → "Tenant Admin", otherwise → "Tenant Member".

### Tenant onboarding

- A tenant must be explicitly registered in UCP before provisioning is allowed.
  Registration is open to any OC Tenant Admin for their tenant — no pre-existing
  UCP role required.
- On registration, all OC Tenant Admins for that tenant who already have a UCP
  account are automatically assigned `tenant-admin`. Those who have not yet
  logged in receive it on their first login.
- All OC tenant members are stored locally (including those who have never logged
  in to UCP), so the member picker shows the full OC roster without live Horizon
  calls.

### Horizon Core Data API findings

Testing all relevant Horizon endpoints confirmed:
- `subscriptions[].default_role` is absent for most services — not a reliable
  source for per-member service roles.
- `GET /v0/members/{rns}/tenants/{rns}/services/{rns}/access/roles?verify` is
  the only reliable real-time service role check. This is what Option 2 would
  use at provisioning time if adopted.
- The user's own JWT is sufficient for all Horizon calls — no platform-level
  service account is required.

See [Horizon Core Data API](./horizon-core-data-api.md) for full test results.

---

## 4. Open Questions


1. **JWT `groups` format for Tenant Members** — does `rns:roc:iam::{tenant}:roles:member`
   appear for OC Tenant Members, or do they simply have no `iam` group entry?
   Needs a non-admin test account. If confirmed, the `admins[]` cross-reference
   step can be eliminated — OC role becomes deterministic from each user's own
   JWT on login. `GET /v0/tenants/{rns}/members` is still needed to discover all
   members regardless.

2. **Can a tenant-admin demote another OC Tenant Admin?** — revoking a UCP role
   from an OC Tenant Admin is transient; the login sync re-grants it on their
   next login. Should revocation be blocked, or should there be a way to
   permanently suppress a sync-granted role?

3. **OC tenant membership as user identity (single vs separate table)** — should
   OC tenant members be pre-provisioned as user records with no login activity,
   enabling immediate role assignment? Or keep the current two-table approach
   (authenticated accounts separate from the full OC member snapshot)?
   Trade-off: simpler role assignment model vs storing users who may never log in.

4. **Non-user member types** — the OC member list includes `service-account` and
   `team` types (confirmed in testing). Should these be excluded from role
   assignment? Should the API block role assignment for non-user types?

5. **Pre-login role assignment** — OC Tenant Admins who have not yet logged in
   to UCP cannot be pre-assigned a UCP role. They receive it automatically on
   first login. Whether this gap is acceptable for MVP or whether pre-provisioning
   is required needs a decision (related to Open Question 3).

6. **Periodic background sync** — the current implementation syncs on login only.
   Role changes in OC between logins are not reflected until the user next logs in.
   Whether this staleness window is acceptable for MVP, or whether a background
   sync job is required, needs a decision.


7. **Cloud provider credential and authorization strategy** — UCP cannot federate
   into each cloud provider's native authz system using Keycloak JWTs without
   provider-specific configuration that does not generalize across providers.
   The PoC uses one SA per provider per tenant with UCP as the security boundary,
   which is the right foundation. The open question is how granular UCP's own
   authz layer should be on top of that:

   - **Tenant-level roles only (current PoC)** — `developer`, `tenant-admin`,
     `platform-admin` apply to all resource types equally
   - **UCP-native service-level roles** — UCP introduces per-resource-type role
     assignments (e.g. `developer` for databases, `viewer` for VMs), independent
     of OC service roles
   - **OC service roles applied cross-provider** — a user's OC service role
     (e.g. `dbaas:admin`) grants corresponding rights in UCP for that resource
     type across all providers (Omnia DBaaS, GCP Cloud SQL, AWS RDS all treated
     as `database`). Data already collected from JWT; no new CLI commands needed
     for role management

   The OC service roles approach is the lowest-friction path to finer-grained
   control and keeps OC as the source of truth. The data is already in
   `oc_roles.oc_service_roles`; enabling it is a middleware change only.
   Decision deferred to PM.

   See [Cloud Provider Authorization Model](../../research/cloud-provider-authz-model.md)
   for full analysis including STRIDE threat model and comparison table.

---

## 5. Recommendations

**Decision: Pending**

The RBAC model, tenant onboarding, and member sync are functionally complete.
The permission bitmask is extensible without schema changes. The local-first
sync keeps request latency low with Horizon dependency confined to login time.


**Next steps:**

1. Verify JWT `groups` format for Tenant Members with a non-admin OC account
   (Open Question 1)
2. Decide on single vs separate table for OC member identity before MVP
   (Open Question 3) — impacts schema migration scope
3. Decide on pre-login role assignment and background sync approach for MVP
   (Open Questions 5, 6)
4. Formally review and decide on cloud provider credential strategy before MVP
   (Open Question 7)

---

## 6. References

- Design docs: [RBAC — UCP Design](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6655504986/RBAC)
- PM requirements: [UCP Identity, Tenancy & Roles](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6645566515/UCP+Identity+Tenancy+Roles)
- Horizon Core Data API: [horizon-core-data-api.md](./horizon-core-data-api.md)
- PRs: [ucp-platform #78](https://ghe.rakuten-it.com/clsd-ucp/ucp-platform/pull/78) · [ucp-api-gateway #26](https://ghe.rakuten-it.com/clsd-ucp/ucp-api-gateway/pull/26) · [ucp-ui #20](https://ghe.rakuten-it.com/clsd-ucp/ucp-ui/pull/20)
- Jira: [MCUCP-191](https://jira.rakuten-it.com/jira/browse/MCUCP-191)
- Prerequisite: [MCUCP-192 — Multi-tenancy](https://jira.rakuten-it.com/jira/browse/MCUCP-192)
