---
title: "GCP Quota Concepts and Glossary"
space: UCP
parent_page_id: "../quota-management.md"
---

# GCP Quota Concepts and Glossary

---

## Quota vs Limit

**Quota** — a soft upper bound on how much of a resource a project can consume. Adjustable
via a support request. GCP may auto-approve small increases.

**Limit** — a hard, fixed ceiling imposed by the service architecture. Cannot be changed
regardless of support tier or justification.

GCP documentation: *"Quotas are adjustable. Limits are not."*

Examples:
- "Your project can use up to 24 CPUs in `us-central1`" — **quota** (can be raised)
- "A Cloud SQL instance can have at most 96,000 connections" — **limit** (architectural max)
- "A single Pub/Sub message cannot exceed 10 MB" — **limit**

---

## Types of Quotas

### Allocation quota (resource quota)

A limit on **how many instances of a resource can exist at one time**. Checked at resource
creation or reservation time. Releasing the resource returns the quota.

Unlike rate quotas, allocation quotas do not reset on a time window. If you are at the
limit, you must delete an existing resource before creating a new one.

Examples: CPUs in use per region, Cloud SQL instances per project, GKE clusters.

This is the type UCP cares most about for pre-provision checks.

### Rate quota (API call quota)

A limit on **how frequently you can call a management API**, measured in requests per unit
of time. Exceeding a rate quota means your API call was rejected because you sent too many
requests too fast — not because you have too many resources. Resets after the window.

**Important for UCP:** using a Cloud SQL database — running SQL queries, reading/writing
data — does **not** count against rate quotas. Database traffic goes directly over a TCP
connection to the database instance. It never passes through the Admin API. The Cloud SQL
`connect` rate quota specifically measures calls to `GetConnectSettings` and
`GenerateEphemeralCertificate` (Auth Proxy connection setup), not SQL queries.

Rate quotas matter for operations teams monitoring API call volume. They are generally
irrelevant for UCP tenants managing a handful of resources.

---

## Scope

| Scope | Description | Example |
|-------|-------------|---------|
| **Global** | Entire project regardless of region | Total global static IP addresses |
| **Regional** | Per region, tracked independently | CPUs in `us-central1` vs `europe-west1` |
| **Zonal** | Rare; some GPU types | — |
| **Project-level** | Default; each project has independent quota | All of the above |

There is no cross-project quota pooling by default. A project's quota is entirely
independent of other projects in the same billing account or organisation.

---

## Quota Metric

The unique identifier for a specific quota in GCP's quota system.
Format: `{service-domain}/{resource-name}`

Examples:
- `sqladmin.googleapis.com/connect` — Cloud SQL Admin API connect rate quota
- `compute.googleapis.com/cpus` — Compute Engine CPU allocation quota
- `container.googleapis.com/clusters` — GKE cluster count quota

Not every GCP limit has a quota metric. Soft limits (like Cloud SQL instance count) do not.

---

## Dimension

A qualifier that scopes a quota metric to a specific context such as a region or zone.
A single quota metric can have different limit values for different dimensions.

Example: `compute.googleapis.com/cpus` has a separate limit row per GCP region. Each row
can have a different limit value. This is why a single service can produce hundreds of rows
in the quota table.

---

## Hard Quota vs Soft Limit (programmatic availability)

| Class | Limit via quota API | Usage via Cloud Monitoring | Example |
|-------|--------------------|-----------------------------|---------|
| Hard quota with metric ID | Yes | Yes (where supported) | Compute CPUs, GKE clusters |
| Soft limit without metric ID | **No** | Partial (count only, no ceiling) | Cloud SQL instances per project |

Cloud SQL instances per project is a **soft limit** managed via support cases. It does
not appear in the Cloud Quotas API or Cloud Monitoring. This blocks the pre-provision gate
for databases.

---

## Default Quota Values (reference)

### Compute Engine (per region unless noted)

| Quota | Default |
|-------|---------|
| CPUs (N1, N2, E2) | 24 |
| Preemptible / Spot CPUs | 24 |
| GPUs (all types) | **0** — must request, can take a week+ |
| Static IP addresses | 8 |
| Persistent Disk SSD (GB) | 500 |

### GKE

| Quota | Default |
|-------|---------|
| Clusters per zone | 50 |
| Clusters per region | 50 |
| Nodes per cluster (hard limit) | 15,000 |

### Cloud SQL

| Quota | Default | Type |
|-------|---------|------|
| Instances per project | 100 | Soft limit — no metric ID |
| vCPUs per instance (hard limit) | 96 max | Hard limit |

### Cloud Storage

| Quota | Default |
|-------|---------|
| Buckets per project | 100 |

---

## How Quota Errors Surface

### HTTP

| Scenario | HTTP Status | `reason` |
|----------|-------------|----------|
| Rate limit exceeded | 429 | `rateLimitExceeded` |
| Allocation quota exceeded | 429 (newer) or 403 (older) | `quotaExceeded` |

### Error response body

```json
{
  "error": {
    "code": 429,
    "status": "RESOURCE_EXHAUSTED",
    "errors": [{ "domain": "usageLimits", "reason": "quotaExceeded" }],
    "details": [{
      "@type": "type.googleapis.com/google.rpc.QuotaFailure",
      "violations": [{ "subject": "project:my-project", "description": "Quota exceeded for CPUS in region us-central1" }]
    }]
  }
}
```

**Distinguishing the two:**
- `rateLimitExceeded` → retry with exponential backoff
- `quotaExceeded` → must release resources or request increase; retrying immediately will not help

---

## Organisation Policies vs Quotas

These are two entirely separate systems:

| Mechanism | What it controls | Numeric limit | Hierarchy-inherited |
|-----------|-----------------|---------------|---------------------|
| Per-project quota | Max resource quantity | Yes | No |
| Org quota policy (enterprise) | Preferred/max quota for projects in folder/org | Yes | Yes |
| Org policy constraint | Which resource types/configs are allowed (allow/deny) | No | Yes |

Org policy constraints (e.g. `constraints/gcp.resourceLocations`) restrict *what* you
can create. A region-blocking constraint makes that region's quotas irrelevant, but does
not itself set a numeric limit.

New projects always start at GCP defaults regardless of org quota levels. High quotas in
a parent org do not automatically propagate to new projects.

---

## Quota Alerts vs Budget Alerts

| | Quota Alert | Budget Alert |
|--|-------------|--------------|
| Measures | Resource consumption (% of quota limit) | Money spent |
| Configured in | Cloud Monitoring (Alerting) | Billing (Budgets & Alerts) |
| Stops resource creation? | No | No |
| Resets | As usage decreases | Monthly |

These are completely separate systems. A budget alert does not prevent quota exhaustion.
Hitting a quota limit does not trigger a budget alert.
