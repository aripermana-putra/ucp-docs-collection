---
jira_ticket: MCUCP-191
title: Role-Based Access Control (RBAC) + Tenant Onboarding
author: aripermana.putra
date: 2026-06-01
status: COMPLETED
---

## 1. Summary

MCUCP-191 proves that UCP can enforce per-tenant role-based access control and
automate tenant onboarding from OC Core Data. A 3-role permission model
(`developer`, `tenant-admin`, `platform-admin`) using a bitmask design is
deployed across all API endpoints and the frontend. Tenant membership and OC
roles are synchronised from Horizon on every login — storing all members locally
so that role management requires zero live Horizon calls at request time.

**Verdict: Go.** The permission model, onboarding flow, and member sync all
work end-to-end. Several open questions remain (see Section 4) that should be
resolved before production hardening.

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
| SC-10 | `platform-admin` can access `/workflows` and cross-tenant resources | Pass |

**Scope boundaries (out of scope):**
- Keycloak configuration changes
- OC service-level role enforcement at provisioning time (Option 2)
- Pre-assigning UCP roles to members who have never logged in
- Periodic background sync between logins

---

## 3. Findings

### Permission model

- A **Permission bitmask** (`PermRead | PermProvision | PermApprove | PermManage | PermPlatform`)
  correctly separates `developer` and `tenant-admin` — a `developer` cannot approve
  their own provisioning request since `PermApprove` is absent from their bitmask.
- The middleware resolves the tenant context from either `?tenantId=` (GET) or
  `{tenantSlug}` (mutations) before the DB role fetch. This enables a targeted
  `WHERE tenant_rns IN ($rns, '*')` query — at most 2 rows regardless of how many
  tenants a user belongs to.

### Login sync

- The Keycloak JWT `groups` claim encodes ALL of the logged-in user's OC tenant
  memberships and per-service roles (format: `rns:roc:{service}::{tenant}:roles:{role}`).
  This allows deriving the user's own tenant list and roles without any Horizon call.
- OC role for other members cannot be reliably read from `GET /v0/tenants/{rns}/members`
  (the `role` field is empty in practice). The correct approach is to cross-reference
  against the `admins[]` array from `GET /v0/tenants/{rns}` — email in admins = "Tenant Admin",
  otherwise "Tenant Member".
- Making the sync **synchronous** (not a goroutine) was necessary: the tenant page
  calls `GET /api/v1/me/tenants` immediately on mount, and a race condition caused
  an empty member list when the page loaded before the async sync completed.

### Tenant onboarding

- `oc_tenant_members` (no FK to `users`) is the correct design for storing all OC
  members regardless of UCP login status. Separating this from `oc_roles` (which
  requires a `users` FK) allows the member picker to show all OC members including
  those who have never logged in.
- `GET /api/v1/me/tenants` reading from local DB (zero Horizon calls) makes the
  tenant page load fast and predictable. Admin contact info for States B/D comes
  from `oc_tenant_members` rather than a live Horizon call.
- Resource menus on the sidebar correctly hide when no tenant is registered — the
  login sync only assigns `tenant-admin` for **registered** tenants, so a new user
  sees only the Tenants menu until they complete onboarding.

### Notable behaviours / surprises

- Horizon's `/members` endpoint returns an empty `role` field — the role must be
  derived from the `/tenants/{rns}` `admins[]` array. This was only discovered during
  live testing.
- The `tenants` table (Yusuke's code) may not exist when UCP is first deployed.
  The fallback `ResolveTenantRNSBySlug` using `oc_roles` handles this gracefully.
- The Keycloak JWT `groups` format for Tenant Members (vs Tenant Admins) is assumed
  to be `rns:roc:iam::{tenant}:roles:member` but could not be verified with a
  non-admin test account.
- OC Tenant Admins who haven't logged in yet cannot be pre-assigned a UCP role
  because `tenant_role_assignments` has a FK to `users(id)`. They receive the role
  automatically on first login via `seedOwnRolesFromJWT`.

---

## 4. Open Questions

**Needs more investigation:**

1. **JWT `groups` format for Tenant Members** — does `rns:roc:iam::{tenant}:roles:member`
   appear for OC Tenant Members, or do they simply have no `iam` group entry?
   Needs a non-admin test account. If confirmed, the Horizon `/members` call can
   be eliminated entirely.

2. **Can a tenant-admin demote another OC Tenant Admin?** — revoking a UCP role
   from an OC Tenant Admin is transient; the sync re-grants it on next login.
   Should `RevokeRole` block this, or should there be a "permanently suppress"
   flag?

3. **OC tenant membership as user identity (single vs separate table)** — should
   OC tenant members be pre-provisioned into the `users` table with `last_login_at = null`,
   enabling immediate role assignment? Or keep the current two-table approach
   (`users` for authenticated accounts, `oc_tenant_members` for the full OC snapshot)?
   Trade-off: simpler FK model vs table bloat with members who never use UCP.

4. **Non-user member types** — `oc_tenant_members.member_type` stores `"user"`,
   `"service_account"`, `"team"`, etc. Should service accounts and teams be excluded
   from the role assignment UI? Should `AssignRole` block non-user types? Exact
   type values from Horizon are unverified beyond `"user"`.

**Remains open:**

5. **OC service-level role enforcement (Option 2)** — the `oc_roles.oc_service_roles`
   JSONB column already stores per-service OC roles from the JWT `groups` claim.
   Enabling runtime OC service-role checks at provisioning time requires only a
   middleware wire-up — no new data collection. Decision deferred to PM.

6. **Public cloud service roles** — OC defines service-level roles (DBaaS admin/operator/viewer)
   but GCP has no equivalent OC service-role concept. Currently a `developer` can
   provision any GCP resource the tenant's credentials cover. Whether to introduce
   per-service UCP roles is an open design question for the PM.

---

## 5. Recommendations

**Decision: Go**

The RBAC model, tenant onboarding, and member sync are functionally complete for
the PoC. The permission bitmask architecture is future-proof — adding a new role
is a one-line change in `RolePermissions`. The local-first sync approach keeps
request latency low and Horizon dependency confined to login time.

**Critical risks:**

| Risk | Mitigation |
|---|---|
| Horizon `admins[]` cross-reference may break if OC changes response shape | Detect empty member list after sync; surface error to admin |
| Sync failure silently leaves members without UCP roles | All sync errors logged; `seedOwnRolesFromJWT` ensures at minimum the logged-in user is seeded |
| Pre-login OC Tenant Admins don't get `tenant-admin` at registration | They receive it on first login — gap is acceptable for PoC |
| `tenants` table may be absent (Horizon sync not run) | Fallback `ResolveTenantRNSBySlug` from `oc_roles` handles this |

**Next steps:**

1. Verify JWT `groups` format for Tenant Members with a non-admin OC account
   (Open Question 1) — if confirmed, remove the Horizon `/members` call
2. Decide on single vs separate table for OC member identity (Open Question 3)
   before production — impacts schema migration scope
3. Align with PM on whether to enable Option 2 (OC service-role check at
   provisioning) and non-user member type filtering (Open Questions 5, 6)
4. Periodic sync as background job — deferred; currently relies on login-triggered
   sync. Acceptable for PoC but should be addressed for production

---

## 6. References

- Design docs: [RBAC — UCP Design](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6655504986/RBAC)
- PM requirements: [UCP Identity, Tenancy & Roles](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6645566515/UCP+Identity+Tenancy+Roles)
- PRs: [ucp-platform #78](https://ghe.rakuten-it.com/clsd-ucp/ucp-platform/pull/78) · [ucp-api-gateway #26](https://ghe.rakuten-it.com/clsd-ucp/ucp-api-gateway/pull/26) · [ucp-ui #20](https://ghe.rakuten-it.com/clsd-ucp/ucp-ui/pull/20)
- Jira: [MCUCP-191](https://jira.rakuten-it.com/jira/browse/MCUCP-191)
- Prerequisite: [MCUCP-192 — Multi-tenancy](https://jira.rakuten-it.com/jira/browse/MCUCP-192)
