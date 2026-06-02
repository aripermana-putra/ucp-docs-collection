---
title: "Multi-Tenant Quota Strategies"
space: UCP
parent_page_id: "../quota-management.md"
---

# Multi-Tenant Quota Strategies for GCP

## The Two Fundamental Models

### Option A — Project Per Tenant

Each tenant gets a dedicated GCP project. GCP projects are the native quota isolation
boundary — every quota is enforced per-project independently.

**Pros:**
- Hard quota isolation by design — one tenant exhausting their quota has zero effect on others
- Billing clarity — each project maps cleanly to billing with no label engineering
- IAM isolation — project boundaries are also IAM boundaries
- Scoped quota increases — support cases are isolated per tenant

**Cons:**
- Operational overhead scales linearly with tenant count — project creation, billing linkage, IAM, VPC, API enablement, quota bootstrapping must be automated
- New projects start at GCP defaults — quota increase requests can take days for large amounts

### Option B — Shared Project

All tenants share one GCP project. The platform enforces resource limits at the API layer.

**Pros:**
- Simpler project management
- Quota increases benefit all tenants at once

**Cons:**
- No hard GCP-level quota isolation — a bug in the platform quota ledger or resources created outside the platform can exhaust shared quota and affect all tenants
- At scale, a single project accumulates extremely large resource counts causing API latency
- Billing chargeback requires label discipline and is error-prone

### Recommendation

**For UCP: Project per tenant.** UCP already implements ProviderConfig per tenant per
provider, and each ProviderConfig points to a dedicated cloud account/project. This is
consistent with Option A and provides hard GCP-level isolation by default.

---

## Platform Quota Enforcement Patterns

Once tenant isolation is in place, the platform soft quota can be enforced in three ways:

| Approach | Latency | Consistency | Complexity |
|---|---|---|---|
| API pre-flight check | Synchronous | Moderate (TOCTOU risk at high concurrency) | Low |
| Controller reconcile gate | Async | Eventually consistent | Low–Medium |
| Kubernetes admission webhook | Synchronous | Strong | High |

**API pre-flight check** is the current UCP design intent — the API server checks the
quota ledger before creating an XR. Simplest to implement and sufficient for the expected
concurrency at MVP.

Platform soft quotas should be set below the GCP project quota to leave headroom —
preventing any tenant from accidentally exhausting the GCP-level limit.
