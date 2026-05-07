---
title: "Quota Management"
space: UCP
parent_page_id: "../pocs.md"
---

# Quota Management

## Overview

This document covers research into cloud provider quota management, focusing on GCP,
with the goal of understanding what UCP needs to implement for proper multi-tenant
quota enforcement.

**Key finding:** UCP has a complete quota design in `docs/architecture/RBAC.md` (Phase 2),
but zero implementation exists. Tenant isolation (MCUCP-119) is a prerequisite —
quota enforcement only makes sense once resources are correctly scoped per tenant.

---

## What Is Quota Management?

In a multi-tenant platform, quota management serves two distinct purposes:

1. **Cloud provider quota** — GCP enforces hard limits on how many resources a project
   can consume (CPUs per region, Cloud SQL instances, etc.). These are set by Google,
   defaults are conservative, and increases must be requested and approved.

2. **Platform quota** — The UCP platform enforces soft limits per tenant before touching
   GCP at all. This prevents one tenant from exhausting shared GCP quota and starving
   others, and gives the platform independent control over resource allocation regardless
   of what GCP allows.

These two layers interact but are managed separately.

---

## UCP Current State

| Component | Status |
|---|---|
| Quota design specification | Designed — `docs/architecture/RBAC.md` Phase 2 |
| `quota_policies` DB table | Not implemented |
| `CheckQuota` middleware | Not implemented |
| Quota API endpoints | Not implemented |
| Frontend quota UI | Not implemented |
| K8s ResourceQuota/LimitRange objects | Not deployed |
| XRD quota fields | Not defined |

The RBAC.md specification is production-ready and should be used as the implementation
guide. See the UCP Quota Design doc for the full design and gap analysis.

---

## Relationship to Tenant Isolation

Quota enforcement only works correctly once tenant isolation is in place. Specifically:

- The `CheckQuota` middleware counts resources via the K8s API — this requires resources
  to be labeled with `ucp.platform/tenant` (Gap 2 from tenant isolation work).
- The list query used for counting must filter by tenant label (Gap 1 from tenant
  isolation work).
- Without these, a quota of "5 databases" would count all tenants' databases, not
  just the requesting tenant's.

**Sequence:** fix tenant isolation gaps first → then implement quota enforcement.

---

## Sub-Documents

- [GCP Quota Concepts](quota-management/gcp-quota-concepts.md) — quota vs limit, types,
  defaults, error surfacing, billing effects
- [GCP Quota APIs](quota-management/gcp-quota-apis.md) — Cloud Quotas API, ServiceUsage
  API, gcloud CLI, Cloud Monitoring metrics, Terraform, Crossplane gap
- [Multi-Tenant Quota Strategies](quota-management/multi-tenant-strategies.md) —
  project-per-tenant vs shared project, platform vs cloud enforcement, quota as code,
  common pitfalls, landing zone patterns
- [UCP Quota Design](quota-management/ucp-quota-design.md) — existing design,
  implementation gaps, recommended path

---

## Related

- `docs/architecture/RBAC.md` — Phase 2: Quota Policies (full design specification)
- `pocs/tenant-isolation.md` — prerequisite work (tenant labels, list filtering)
- `docs/ARCHITECTURE_VERIFICATION_REPORT.md` — Gap #16: Quota Management
