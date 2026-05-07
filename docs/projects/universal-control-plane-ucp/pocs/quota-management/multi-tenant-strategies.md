---
title: "Multi-Tenant Quota Strategies"
space: UCP
parent_page_id: "../quota-management.md"
---

# Multi-Tenant Quota Strategies for GCP

## The Two Fundamental Models

### Option A — Project Per Tenant

Each tenant gets one or more dedicated GCP projects. GCP projects are the native quota
isolation boundary — every quota is enforced per-project independently.

```
Tenant A → GCP Project: team-a-prod → GCP quota bucket A (CPUs, SQL instances, etc.)
Tenant B → GCP Project: team-b-prod → GCP quota bucket B (independent)
```

**Pros:**
- **Hard quota isolation by design.** Tenant A exhausting their CPU quota has zero
  effect on Tenant B. GCP enforces this at the API layer — no platform code needed.
- **Billing clarity.** Each project maps cleanly to billing. Chargeback is
  straightforward without any label engineering.
- **IAM isolation.** Project boundaries are also IAM boundaries — a service account in
  project A cannot access project B without an explicit binding.
- **Scoped quota increases.** When a tenant needs more vCPUs, only their project is
  involved. The support case is isolated.
- **Org policy scoping.** Per-tenant compliance constraints can be set at the project
  level without affecting others.

**Cons:**
- **Operational overhead scales linearly.** 50 tenants = 50+ projects, each needing
  billing linkage, IAM setup, VPC configuration, API enablement, quota baseline, and
  audit log routing. Automation (project factory) is mandatory.
- **Project limits.** GCP allows up to 1,000 active projects per billing account by
  default (increasable). Organisations with thousands of tenants must plan for this.
- **Quota bootstrapping delay.** New projects start with conservative defaults.
  Quota increase requests can take 2–3 business days for large amounts. The platform
  cannot provision full capacity for a new tenant immediately.
- **Cross-tenant networking complexity.** Shared services require Shared VPC, VPC
  peering, or Private Service Connect.

### Option B — Shared Project

All tenants share one (or a few) GCP projects. The platform enforces resource limits at
the API layer before requests ever reach GCP.

```
All tenants → GCP Project: ucp-shared-prod → Platform-level soft quotas per tenant
```

**Pros:**
- Simpler project management — one project to operate.
- Quota increase requests benefit all tenants at once.
- Cross-tenant resource sharing (shared VPC, shared GKE cluster) is natural.
- Lower bootstrap overhead for new tenants.

**Cons:**
- **No hard GCP-level quota isolation.** A bug in the platform's quota ledger, or
  resources created outside the platform, can exhaust shared quota and starve all tenants.
- **Billing chargeback** requires labels and BigQuery query discipline — more error-prone.
- At scale, a single project accumulates extremely large resource counts, which causes
  API list operation latency and project-level ceilings.
- Difficult to apply per-tenant org policies.
- A compromised tenant service account potentially sees other tenants' resource metadata.

### Recommendation Matrix

| Scenario | Recommended Model |
|---|---|
| <20 tenants, strong isolation requirement | Project per tenant |
| <20 tenants, cost priority, trusted teams | Shared project with soft quotas |
| 20–200 tenants, IDP at scale | Project per tenant with automated factory |
| >200 tenants with varying sizes | Mixed: shared project for small/dev, dedicated for large/prod |
| GPU/ML heavy workloads | Project per tenant always |
| Strong chargeback requirement | Project per tenant |
| Showback only (informational) | Shared project with labels |

**For UCP (multi-cloud IDP, internal tenants):** Project per tenant is the right model.
UCP already implements ProviderConfig per tenant per provider, and each ProviderConfig
points to a dedicated cloud account/project. This is consistent with the project-per-tenant
approach.

---

## Organisation-Level Governance

### Resource Hierarchy

```
Organisation
├── bootstrap/         (Terraform state, automation service accounts)
├── common/            (shared VPC host project, logging, billing sinks)
├── nonprod/
│   └── tenants/       (team-a-dev, team-b-dev)
└── prod/
    └── tenants/       (team-a-prod, team-b-prod)
```

**What folders provide:**
- Org policy inheritance (policies at `prod/tenants/` apply to all tenant prod projects)
- Folder-level quota caps via Cloud Quotas API (enterprise feature, 2024)
- IAM inheritance for platform service accounts
- Billing scoping

**What folders do NOT provide:**
- Aggregated quota pools across projects — each project still has its own independent
  quota values
- Cross-project resource access controls

### Org Policy Baseline (Applied at Org/Folder Level)

```
constraints/compute.skipDefaultNetworkCreation = true
  → Prevents auto-created default VPC; forces explicit VPC setup

constraints/iam.disableServiceAccountKeyCreation = true
  → Forces Workload Identity instead of long-lived key files

constraints/compute.vmExternalIpAccess = deny all
  → No public IPs by default; tenants must explicitly request exceptions

constraints/gcp.resourceLocations = restrict to approved regions
  → Prevents accidental resource creation in non-approved regions
```

---

## Platform vs Cloud Quota Enforcement

### Cloud Layer (Hard Quotas)

GCP enforces quotas at the API gateway. A `RESOURCE_EXHAUSTED` (HTTP 429) terminates
the request immediately. No override at runtime. Applies per-project, per-service,
per-region.

### Platform Layer (Soft Quotas)

Platform enforces soft quotas before GCP API calls. Three patterns:

**1. API pre-flight check** (simplest, current UCP design intent)

The REST API server checks a quota ledger before calling `kubectl apply` or making a
GCP API call. If tenant A has a policy of 5 Cloud SQL instances and currently has 5,
the API returns 429 before creating the XR.

Risk: TOCTOU (time-of-check/time-of-use) race condition if resources are created through
multiple code paths simultaneously.

**2. Controller reconcile gate** (most natural for Crossplane)

The Crossplane composition controller or a custom controller checks per-tenant quota
state in its reconcile loop. If quota is exceeded, it sets a `QuotaExceeded` condition
on the XR and does not proceed. Eventually consistent but avoids TOCTOU.

**3. Kubernetes admission webhook** (strongest enforcement)

A validating webhook intercepts XR creation. Before a claim is accepted, the webhook
checks the quota policy and rejects if the limit is reached. Synchronous and strong,
but requires webhook cert management and high availability.

| Approach | Latency | Consistency | Complexity |
|---|---|---|---|
| API pre-flight | Synchronous | Moderate (TOCTOU risk) | Low |
| Controller reconcile gate | Async | Eventually consistent | Low–Medium |
| Admission webhook | Synchronous | Strong | High |

**Soft vs Hard Quota Relationship:** Soft quotas at the platform layer should be set
*below* the actual GCP project quota to leave headroom. Example: if the project has 240
CPUs, platform enforces a soft cap of 40 CPUs per tenant for 6 tenants. This prevents
any single tenant from exhausting the project quota.

---

## Quota as Code

### Pattern: Tenant Entitlement Files

Declare per-tenant quota entitlements in a Git repository (GitOps model):

```yaml
# entitlements/team-analytics.yaml
tenant: team-analytics
environment: prod
entitlements:
  compute:
    gce_vcpus_us_central1: 128
    gke_clusters: 3
  database:
    cloudsql_instances: 5
    cloudsql_vcpus: 16
  storage:
    gcs_buckets: 20
  networking:
    external_ips: 3
    forwarding_rules: 10
```

This file drives two things:
1. The platform's admission controller enforces these as soft caps
2. A reconciler compares entitlements to current GCP quota and opens quota increase
   requests via Cloud Quotas API when the entitlement exceeds the current GCP limit

### Pattern: TenantQuota CRD

For Crossplane-based IDPs, model quotas as a Kubernetes CRD:

```yaml
apiVersion: platform.io/v1alpha1
kind: TenantQuota
metadata:
  name: team-analytics
  namespace: team-analytics
spec:
  gce:
    vcpusPerRegion:
      us-central1: 128
  cloudsql:
    instances: 5
  gke:
    clusters: 3
status:
  gcpActualQuota:
    gce.vcpus.us-central1: 96    # what GCP currently allows
  conditions:
    - type: QuotaAligned
      status: "False"
      message: "GCP quota (96) below entitlement (128), increase pending"
```

A controller watches `TenantQuota` objects. When `spec > status.gcpActualQuota`, it
calls the Cloud Quotas API to submit a `QuotaPreference`. The `QuotaAligned` condition
becomes `True` when GCP approves and the project's actual quota matches the entitlement.

### Key Distinction: Override vs Increase Request

These are two different operations with different response characteristics:

| Operation | API | Direction | Response time | Use case |
|---|---|---|---|---|
| Consumer quota override | ServiceUsage API | Downward only (cap) | Immediate | Prevent tenant from using all granted quota |
| Quota increase request (QuotaPreference) | Cloud Quotas API | Upward | Async, 0–3+ days | Request more quota from GCP |

A complete quota-as-code system must handle both operations.

---

## Chargeback and Showback

### Project Per Tenant (simplest)

GCP Billing export to BigQuery provides exact per-project cost data with no label
engineering. Each project maps to billing cleanly.

```sql
SELECT project.id, SUM(cost) as total_cost
FROM billing_export.gcp_billing_export_v1_XXXXXX
WHERE invoice.month = '202503'
GROUP BY project.id
```

### Shared Project (label-based)

Requires every GCP resource to be tagged with a `team` label at creation time. Problems:
- Some GCP services do not pass resource labels to billing entries
- Labels can be forgotten or inconsistently applied
- Network egress and cross-service traffic are hard to attribute

The industry pattern: showback (informational) first, chargeback (financial allocation)
only when billing accountability has executive sponsorship.

### Quota as a Budget Proxy

Many platform teams use quota as a **budget proxy**: instead of enforcing a dollar
budget (complex, requires real-time billing pipeline), they set quota limits that
indirectly cap spend. Example: limiting a tenant to 40 vCPUs caps their compute spend
at ~$2,800/month in `us-central1`. Simpler than real-time billing enforcement.

---

## Common Pitfalls

### Quota Exhaustion in Shared Projects

In the shared project model, one tenant's burst workload can exhaust the project's
regional quota and cause all other tenants' provisioning requests to fail with 429.

**Mitigation:** Alert at 70% quota utilisation, apply per-tenant soft caps at the
platform layer, and pre-emptively request quota increases before hitting the limit.

### Quota Increase Lead Times

GCP quota increases are asynchronous:
- Small increases: often auto-approved within minutes
- Large increases (10x current, GPUs): 2–3 business days, may require justification
- Very large increases: up to a week or more, may require committed use discussion

**Platform implication:** Either pre-warm quota in newly created projects immediately
at project creation (submit increase requests before any tenant requests), or surface
the wait time transparently in the tenant onboarding flow. A pool of pre-provisioned
projects with baseline quota is a common enterprise pattern.

### Regional vs Global Quota Confusion

GCP quotas are mostly regional, not global. A tenant with 100 vCPUs in `us-central1`
cannot use that allowance in `europe-west4`. Entitlement models must declare quotas
per-region, not globally.

### New Service Zero-Default Quotas

When a GCP service is newly enabled on a project, it starts with conservative defaults
or zero for some resources (especially GPUs, Vertex AI, specialised ML hardware).
Always bootstrap quota for anticipated services at project creation time.

### Monitoring Coverage Gap

Not all GCP quotas expose real-time utilisation metrics in Cloud Monitoring. Most common
services are covered via `serviceruntime.googleapis.com/quota/allocation_usage` and
`serviceruntime.googleapis.com/quota/rate/net_usage`, but some service-specific quotas
require their own metric paths. A complete quota monitoring setup requires per-service
metric enumeration.

---

## Project Factory Pattern

For an IDP managing many tenant projects, project creation must be automated.

**Google-maintained Terraform modules:**
- `google-project-factory` — automates project creation, billing linkage, API
  enablement, IAM setup, Shared VPC attachment, default labels
- `cloud-foundation-fabric` — more complete reference implementation by Google PSO;
  includes billing export, log routing, security baseline, and multi-tenant patterns

**Recommended tenant onboarding pipeline:**

```
1. Tenant submits onboarding request (IDP UI, Git PR)
2. Approval workflow validates and obtains approval
3. Project factory creates GCP project, links billing, attaches to Shared VPC,
   applies default org policies
4. Quota bootstrapper immediately submits Cloud Quotas API QuotaPreference requests
   for the tenant's declared entitlements (starts the async approval clock early)
5. Status tracking monitors QuotaPreference state (PENDING → ACTIVE)
6. Tenant environment marked "ready" only when required quota is approved
7. Crossplane XRDs for tenant namespace activated
```

This gives tenants a deterministic experience and makes the quota approval wait
transparent rather than a silent failure.
