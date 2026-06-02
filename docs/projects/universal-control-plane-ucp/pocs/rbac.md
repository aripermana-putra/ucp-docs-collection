---
title: "RBAC"
space: UCP
parent_page_id: "../pocs.md"
---

# RBAC

UCP enforces access control at the API server layer using a
**3-role per-tenant model** (`developer`, `tenant-admin`, `platform-admin`) with a
**permission bitmask**. Each role is a set of permissions rather than a position in a
hierarchy, making the model extensible without schema changes. Role assignments are
stored in UCP's own PostgreSQL database (`tenant_role_assignments` table) — independent
of Keycloak and Horizon.

---

## Approach Summary

Every authenticated request passes through a `RequirePermission(perm)` middleware
before reaching the handler. The middleware resolves the tenant from the request
(`?tenantId=` for GETs, `{tenantSlug}` for mutations), loads the caller's role from the
DB in a targeted query (≤2 rows), and checks whether the role's permission set includes
the required permission. Requests that fail the check are rejected with HTTP 403.

`platform-admin` is a cross-tenant role stored as `tenant_rns = '*'` and passes every
permission check regardless of tenant scope.

OC tenant membership and service roles are synchronised from Horizon on every login and
stored locally — role management at request time requires zero live Horizon calls.

---

## Sub-Documents

- [POC Report](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6667713462/POC+Report+%E2%80%94+RBAC+Tenant+Onboarding) — verdict, success criteria, findings, open questions
- [Concepts](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6655505015/RBAC+%E2%80%94+Concepts) — role definitions, permission bitmask, role
  resolution, RequirePermission middleware, OC roles, tenant onboarding
- [Implementation](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6655505026/RBAC+%E2%80%94+Implementation) — scope, status table, endpoint-role
  mapping, sequence diagrams, verification
- [Design](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6655505032/RBAC+%E2%80%94+UCP+Design) — permission model, role resolution, RequirePermission
  middleware, route registration, login sync, tenant onboarding
- [Horizon Core Data API](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6667713470/RBAC+%E2%80%94+Horizon+Core+Data+API) — member and tenant
  identity endpoints, JWT claim approach, live test results, Option 2 feasibility

---

## Related

- `MCUCP-191` — RBAC + tenant onboarding implementation
- `MCUCP-192` — Multi-tenancy (prerequisite: tenant labels on XRs)
