---
title: "GCP Quota Concepts"
space: UCP
parent_page_id: "../quota-management.md"
---

# GCP Quota Concepts

## Quota vs Limit

These two terms are often confused but are fundamentally different:

**Quota** — a soft upper bound on how much of a resource a project can consume.
Quotas exist to protect GCP infrastructure from unexpected spikes and enable fair
sharing. They are *adjustable* — a project owner can request an increase, and GCP
may approve it automatically (for small increases) or after manual review.

**Limit** — a hard, fixed ceiling imposed by the service itself. Limits cannot be
changed regardless of support tier or justification. They reflect architectural
or system constraints.

GCP documentation states explicitly: *"Quotas are adjustable. Limits are not."*

Concrete examples:
- "Your project can use up to 24 CPUs in `us-central1`" — **quota** (default, can
  be raised to hundreds or thousands with a request)
- "A Cloud SQL instance can have at most 96,000 connections" — **limit** (architectural
  maximum, not negotiable)
- "A single Pub/Sub message cannot exceed 10 MB" — **limit**

---

## Types of Quotas

### By Resource Type

**Allocation quotas (resource quotas)**

Govern the total quantity of a resource that can exist at a point in time. Checked at
resource *creation* or *reservation* time. Releasing the resource returns the quota.

Examples:
- Number of CPUs in use in a region
- Total persistent disk SSD capacity (TB) in a region
- Number of static IP addresses reserved
- Number of Cloud SQL instances per project
- Number of GKE clusters

**Rate quotas (API call quotas)**

Govern how many API requests can be made in a given time window (per second, per minute,
per day). Reset after the window expires.

Examples:
- Compute Engine API requests: 1,200/min
- Cloud Storage JSON API: 1,000 requests/second (reads), 200/second (deletes)
- Pub/Sub publish: 1 GB/s per project

### By Scope

| Scope | Description | Example |
|---|---|---|
| **Global** | Applies to the entire project regardless of region | Total global static IP addresses |
| **Regional** | Per region within a project — most common for Compute | CPUs in `us-central1` tracked separately from `europe-west1` |
| **Zonal** | Rare; some resources tracked per zone | Some GPU types |
| **Project-level** | Default scope; each project has its own independent quota | All of the above |

There is no cross-project quota pooling by default. A project's quota is entirely
independent of other projects in the same billing account or organisation.

---

## Quota Dimensions

GCP quotas are identified along a combination of these dimensions:

| Dimension | Description | Example |
|---|---|---|
| Service | Which GCP API/service | `compute.googleapis.com` |
| Metric | What resource is being measured | `CPUS`, `SSD_TOTAL_GB` |
| Project | GCP project ID | `my-project-123` |
| Region | GCP region | `us-central1` |
| Zone | Specific zone (rare) | `us-central1-a` |

---

## Default Quota Values for Common Resources

### Compute Engine (regional unless noted)

| Quota | Default | Scope |
|---|---|---|
| CPUs (general purpose: N1, N2, E2) | 24 | Per region |
| Preemptible / Spot CPUs | 24 | Per region |
| GPUs (all types: T4, V100, A100) | **0** (must request) | Per region per GPU type |
| Static IP addresses | 8 | Per region |
| In-use IP addresses | 24 | Per region |
| Persistent Disk SSD (GB) | 500 | Per region |
| Persistent Disk Standard (GB) | 2,048 | Per region |

> GPU quotas default to 0 for all new projects. Requires explicit justification and can
> take a week or more to approve for large amounts or specific GPU SKUs.

### GKE

| Quota/Limit | Default | Type |
|---|---|---|
| Clusters per zone | 50 | Quota |
| Clusters per region | 50 | Quota |
| Nodes per cluster | 15,000 | Hard limit |
| Node pools per cluster | 100 | Hard limit |

GKE clusters consume Compute Engine CPU and disk quotas from the same project-level
regional pool as regular VMs.

### Cloud SQL

| Quota/Limit | Default | Type |
|---|---|---|
| Instances per project | 100 | Quota (soft, increasable) |
| vCPUs per instance | 96 max | Hard limit |
| Storage per instance | 64 TB max | Hard limit |

### Cloud Storage

| Quota/Limit | Default | Type |
|---|---|---|
| Buckets per project | 100 | Quota (increasable) |
| Object size | 5 TB max | Hard limit |
| JSON API read requests/sec | 5,000/s per project | Soft |
| JSON API write requests/sec | 1,000/s per project | Soft |

### Networking (VPC)

| Quota/Limit | Default | Type |
|---|---|---|
| VPC networks per project | 15 | Quota |
| Subnets per network | 256 | Quota |
| Firewall rules per network | 200 | Quota |
| Routes per network (static) | 250 | Quota |
| Forwarding rules (regional) | 75 | Quota |
| VPC peering connections per VPC | 25 | Hard limit |

---

## How GCP Enforces Quotas

GCP enforces quotas at the **Service Control API layer** (`servicecontrol.googleapis.com`),
not inside individual services. Every GCP service integrates with this central
infrastructure, which acts as a quota enforcement proxy.

**Enforcement flow for allocation quotas:**

1. Client calls a GCP API (e.g., `POST /compute/v1/projects/{proj}/zones/{zone}/instances`)
2. The API gateway calls Service Control's `AllocateQuota` RPC before allowing the operation
3. Service Control checks current usage against the quota limit for that
   `(project, metric, region)` tuple
4. If available: quota is consumed atomically, operation proceeds
5. If exceeded: Service Control returns `RESOURCE_EXHAUSTED`, the API returns an error

**Key properties:**
- Enforcement is **synchronous** — the API call fails immediately if quota is exceeded
- There is no queuing or deferral — requests are rejected outright
- Regional quotas are tracked independently — exhausting CPUs in `us-central1` does
  not affect `europe-west4`

---

## How Quota Errors Surface

### HTTP Layer

| Scenario | HTTP Status | `reason` field |
|---|---|---|
| Rate limit exceeded (API calls/min) | **429 Too Many Requests** | `rateLimitExceeded` |
| Allocation quota exceeded (resource count) | **429** (newer) or **403** (older APIs) | `quotaExceeded` |

Both codes appear in practice depending on the service and API version.

### Error Response Body

```json
{
  "error": {
    "code": 429,
    "message": "Quota exceeded for quota metric 'compute.googleapis.com/cpus' and limit 'CPUS-per-project-region'...",
    "errors": [{
      "domain": "usageLimits",
      "reason": "quotaExceeded"
    }],
    "status": "RESOURCE_EXHAUSTED",
    "details": [{
      "@type": "type.googleapis.com/google.rpc.QuotaFailure",
      "violations": [{
        "subject": "project:my-project",
        "description": "Quota exceeded for CPUS in region us-central1"
      }]
    }]
  }
}
```

### gRPC Layer

`RESOURCE_EXHAUSTED` = **gRPC code 8**, maps to HTTP 429.

### Distinguishing the two error types

- `rateLimitExceeded` — you are making too many API calls in too short a time. Retry
  with exponential backoff.
- `quotaExceeded` — you have hit the allocated resource quota. You must either release
  resources, request a quota increase, or wait (for time-based windows). Retrying
  immediately will not help.

---

## Quota Inheritance and Organisation Policies

### Quota Does NOT Inherit from Organisation

Each project gets its own independent quota bucket at GCP defaults. Creating a project
under an org with high CPU quotas does not grant the new project higher quotas. New
projects always start with standard defaults.

### Organisation-Level Quota Policies (Enterprise Feature)

Introduced in 2023–2024 via the Cloud Quotas API: org or folder admins can set:

- **Preferred quota** — baseline quota automatically applied to new projects in a
  folder/org
- **Maximum quota** — a cap that prevents projects from ever exceeding a certain value,
  even via increase requests

These are separate from per-project self-service quota management and are aimed at
enterprise governance.

### Organisation Policy Constraints vs Quotas

These are two entirely separate systems:

| Mechanism | What it controls | Numeric limit? | Hierarchy-inherited? |
|---|---|---|---|
| Per-project quota | Max resource quantity per project | Yes | No |
| Quota policy (enterprise) | Preferred/max quota for projects in folder/org | Yes | Yes |
| Org policy constraint | Which resource types/configs are allowed | No (allow/deny) | Yes |

Org policy constraints (e.g., `constraints/gcp.resourceLocations`) restrict *what* you
can create, not *how many*. A constraint blocking a region makes quotas for that region
irrelevant, but it does not itself set a numeric limit.

---

## Billing Account Effects on Quotas

| Account Type | CPU quota (regional) | GPU quota | Can request increase? |
|---|---|---|---|
| No billing account | 0 (most services disabled) | 0 | No |
| Free trial ($300 credit) | 8–12/region (restricted) | 0 | Very limited |
| Paid (standard) | 24/region (default) | 0 (must request) | Yes |
| Paid + established spend | Higher defaults sometimes granted proactively | Requestable | Yes |

GCP proactively increases quotas for projects with established billing history and
consistent usage — this can happen without an explicit request.

---

## Quota Alerts vs Budget Alerts

These are two completely separate systems:

| Dimension | Quota Alert | Budget Alert |
|---|---|---|
| What is measured | Resource consumption (% of quota limit) | Money spent ($ or % of budget) |
| Configured in | Cloud Monitoring (Alerting) | Billing (Budgets & Alerts) |
| Stops resource creation? | No | No (unless wired via Pub/Sub to an action) |
| Useful for | Preventing API 429 errors, planning increases | Cost governance, billing surprises |
| Scope | Per-service, per-region, per-metric | Per-project, per-billing-account |
| Resets | As resource usage decreases | Monthly (or custom period) |
| Related to quota limits? | Yes, directly | No |
| Related to billing? | No | Yes |

A common misconception: setting a budget alert does not prevent quota exhaustion, and
hitting a quota limit does not trigger a budget alert.
