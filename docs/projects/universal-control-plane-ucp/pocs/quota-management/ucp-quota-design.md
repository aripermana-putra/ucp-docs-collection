---
title: "UCP Quota Design"
space: UCP
parent_page_id: "../quota-management.md"
---

# UCP Quota Design

## Current State

Two layers of quota exist. The cloud provider quota layer (GCP) is partially implemented.
The UCP platform soft quota layer is still in the design phase.

### Cloud Provider Quota (GCP) — Partially Implemented

| Component | File | Status |
|---|---|---|
| `QuotaProvider` interface | `backend/api-server/quota_handler.go` | **Implemented** |
| `gcpQuotaProvider` — lists all quota metrics | `backend/api-server/quota_handler.go` | **Deployed** |
| `GET /api/v1/quota` endpoint | `backend/api-server/main.go:197` | **Deployed** |
| Pre-provision gate (`CheckPreProvision`) | `backend/api-server/main.go:478` | **Implemented — fails open** |
| Quota UI | `frontend/src/components/QuotaList.jsx` | **Deployed** |

The pre-provision gate for Cloud SQL databases is blocked: Cloud SQL instance count is a
soft limit with no programmatic metric ID. See `implementation.md` — Finding 2.

### UCP Platform Soft Quotas — Design Phase Only

| Component | Location | Status |
|---|---|---|
| `quota_policies` table schema | `docs/architecture/RBAC.md` §3 | Designed only |
| `CheckQuota` middleware design | `docs/architecture/RBAC.md` §5 | Pseudocode only |
| Quota API endpoints | `docs/architecture/RBAC.md` §7 | Designed only |

Not yet implemented:
- `quota_policies` database table (not in `db/db.go` migrations)
- `CheckQuota()` function
- `/api/v1/rbac/` routes
- RBAC permission tables (`roles`, `role_permissions`, `role_assignments`)
- K8s `ResourceQuota` or `LimitRange` objects
- XRD fields for quota constraints

---

## Existing Design Specification

### Database Schema (from `docs/architecture/RBAC.md`)

```sql
CREATE TABLE quota_policies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_external_id  VARCHAR(256) NOT NULL,      -- Horizon RNS, e.g. rns:roc:iam::clsd-ucp
    resource            VARCHAR(64)  NOT NULL,       -- "database", "compute", "storage", etc.
    limit_value         INTEGER NOT NULL DEFAULT -1, -- -1 = unlimited
    updated_by          UUID REFERENCES users(id),
    updated_at          TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_external_id, resource)
);
```

### CheckQuota Middleware Design (from `docs/architecture/RBAC.md`)

```go
func (a *APIServer) CheckQuota(resource string) mux.MiddlewareFunc {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if r.Method != http.MethodPost {
                next.ServeHTTP(w, r)
                return
            }
            tenantRNS := r.Header.Get("X-Tenant-ID")
            policy, _ := a.db.GetQuotaPolicy(tenantRNS, resource)
            if policy.LimitValue == -1 {
                next.ServeHTTP(w, r) // unlimited
                return
            }
            current, _ := a.countTenantResources(r.Context(), tenantRNS, resource)
            if current >= policy.LimitValue {
                respondError(w, http.StatusTooManyRequests,
                    fmt.Sprintf("quota exceeded: %s limit is %d", resource, policy.LimitValue))
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

Usage count comes from the K8s API (same queries the existing list handlers use),
keeping the API stateless.

### Quota API Endpoints (from `docs/architecture/RBAC.md`)

| Method | Path | Description | Permission |
|---|---|---|---|
| `GET` | `/api/v1/rbac/tenants/{tenantRNS}/quotas` | List all quotas | `quota:read` |
| `PUT` | `/api/v1/rbac/tenants/{tenantRNS}/quotas/{resource}` | Set limit | `quota:update` |
| `GET` | `/api/v1/rbac/tenants/{tenantRNS}/quotas/{resource}/usage` | Current count vs limit | `quota:read` |

---

## Implementation Prerequisites

Quota enforcement depends on tenant isolation being in place first:

### Prerequisite 1 — Tenant Labels on XRs (Tenant Isolation Gap 2)

`countTenantResources()` must filter K8s resources by tenant. This requires the
`ucp.platform/tenant` label to be applied on every XR at creation time.

Without this label, counting "how many databases does tenant A have" is impossible —
the API would count all tenants' databases.

### Prerequisite 2 — Tenant-Scoped List Queries (Tenant Isolation Gap 1)

The K8s list queries used for counting must use a `LabelSelector`:

```go
// Before quota can work:
list, err := s.k8sClient.Resource(gvr).List(ctx, metav1.ListOptions{
    LabelSelector: fmt.Sprintf("ucp.platform/tenant=%s", sanitizeTenantID(tenantRNS)),
})
```

Currently all list handlers use `metav1.ListOptions{}` with no selector (tenant
isolation gap documented in `pocs/tenant-isolation.md`).

### Prerequisite 3 — RBAC Phase 1 (RequirePermission middleware)

Quota API endpoints use `quota:read` and `quota:update` permissions. These require the
RBAC permission model (Phase 1 from `docs/architecture/RBAC.md`) to be implemented first.
Currently only `isUserTenantAdmin()` exists.

**Sequence:**

```
1. Tenant isolation gaps (labels + list filtering)    ← fix first
2. RBAC Phase 1 (roles, permissions, middleware)      ← enables quota API access control
3. Quota Phase 2 (quota_policies table + middleware)  ← quota enforcement
```

---

## Two Layers of Quota: Platform vs GCP

The `quota_policies` table and `CheckQuota` middleware implement **platform-level soft
quotas** — UCP-enforced limits that exist independently of GCP's hard quotas.

To fully manage quotas, UCP needs to handle both:

| Layer | What it controls | Mechanism | Current state |
|---|---|---|---|
| GCP cloud quota | GCP-enforced per-project resource ceiling | Cloud Monitoring (`quota/limit` + `quota/allocation/usage`) | **Partially implemented** — listing works, pre-provision gate fails open |
| Platform soft quota | UCP-enforced per-tenant resource count | `quota_policies` table + `CheckQuota` middleware | Designed only, not implemented |

**Platform quota** prevents a tenant from provisioning more resources than their
entitlement, regardless of GCP limits.

**GCP quota** is a hard ceiling set by Google per project. If the GCP project quota is
exhausted, resource creation fails at the GCP API layer regardless of platform quota.

For UCP with project-per-tenant model (each ProviderConfig points to a dedicated cloud
account), the GCP quota for each tenant project is independent. UCP's platform soft
quota should be set below the GCP quota to leave headroom.

---

## Recommended Implementation Path

### Phase 1 — Platform Soft Quotas (Unblocked After Tenant Isolation)

1. Add `quota_policies` migration to `db/db.go`
2. Implement `db.GetQuotaPolicy()` and `db.SetQuotaPolicy()` in the DB layer
3. Implement `countTenantResources()` using label-filtered K8s list queries
4. Implement `CheckQuota()` middleware
5. Wire `CheckQuota` to all `POST` resource creation handlers:
   - `POST /api/v1/databases` → `CheckQuota("database")`
   - `POST /api/v1/compute` → `CheckQuota("compute")`
   - `POST /api/v1/storage` → `CheckQuota("storage")`
   - `POST /api/v1/kubernetes-clusters` → `CheckQuota("k8s-cluster")`
   - `POST /api/v1/terraform` → `CheckQuota("terraform")`
6. Implement quota CRUD API endpoints (`/api/v1/rbac/tenants/{rns}/quotas/...`)
7. Implement quota usage endpoint (count from K8s, stateless)

### Phase 2 — GCP Cloud Quota Awareness ✅ Partially done

8. ~~Read current GCP quota via Cloud Monitoring~~ — **done** (`GET /api/v1/quota`)
9. Detect when platform entitlement exceeds current GCP quota and surface as a
   condition on the tenant's quota status — **not yet done**
10. Optionally: submit `QuotaPreference` requests via Cloud Quotas API when a
    tenant's entitlement is increased beyond the current GCP limit — **not yet done**

### Phase 3 — Frontend and Quota UI ✅ Partially done

11. ~~Quota display with service filter and usage % bar~~ — **done** (`QuotaList.jsx`)
12. Quota management admin UI (platform-admin and tenant-admin views) — **not yet done**
13. ~~Warning when approaching limit (≥ 80% yellow, 100% red)~~ — **done** (row highlighting)

---

## Open Questions

1. **Who sets initial platform quotas?** Platform-admin via API, or seeded from Horizon
   tenant `tier` field (`standard`/`premium`)? The tier field suggests Horizon already
   encodes capacity intent.

2. **Quota inheritance from org/folder GCP quota policies?** If an org quota policy caps
   a tenant project at 50 CPUs, should UCP read this cap and reflect it as the tenant's
   platform quota automatically?

3. **Cross-provider quota?** Should UCP enforce a single quota across all providers
   (e.g., "5 databases total regardless of whether they are GCP or Omnia"), or per-provider
   quotas (e.g., "5 GCP databases + 5 Omnia databases")?

4. **Quota increase workflow?** Should tenants be able to request a platform quota
   increase via the UCP UI, which then triggers an approval workflow (Temporal) and
   optionally submits a GCP `QuotaPreference`?

5. **Drift between platform quota and GCP quota.** If a resource is deleted directly
   in GCP (not via UCP), the platform ledger count will be wrong. How frequently should
   `countTenantResources()` reconcile against GCP actual state? (This connects to the
   drift detection work — MCUCP-158.)
