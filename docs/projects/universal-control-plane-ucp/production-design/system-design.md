---
title: "System Design"
space: UCP
parent_page_id: "../production-design.md"
---

# UCP Production System Design

---

## 1. The etcd Question

Crossplane runs on exactly one Kubernetes cluster per environment. Each cluster has its
own isolated etcd instance. Crossplane XRs and MRs are Kubernetes objects stored in that
cluster's etcd. The actual cloud resources — GCP Cloud SQL instances, OneCloud databases
— live in cloud provider backends, not in etcd. Temporal workflow state lives in
PostgreSQL, not in etcd.

There is no distributed etcd consistency problem. The topology question is what lives on
which cluster and why — not how to synchronise state between etcd instances.

---

## 2. Non-Functional Requirements

### Given

| NFR | Target |
|---|---|
| API latency | < 1s |
| Drift detection lag | ~1 min |
| Availability | 99.9% (~43.8 min downtime/month) |

### Derived

| NFR | Target | Rationale |
|---|---|---|
| Provisioning acknowledgment latency | < 500ms | Time from POST request to Temporal workflow started. Actual provisioning is async and cloud-provider-bound (20–40 min for databases). |
| Quota check latency | < 200ms | Must fit within the 1s API budget alongside auth and DB lookups. |
| Drift notification delivery | < 5 min from detection | Detection (≤1 min) + DriftApprovalWorkflow start + NotifyDriftActivity execution. |
| Audit log durability | Zero loss | Synchronous write required. No silent drops — 3-year compliance retention demands this. |
| Secret rotation | Zero downtime | Credential refresh without pod restart, via External Secrets Operator syncing from Vault. |
| RTO | < 15 min | Consistent with 3-nines tier and multi-AZ automatic failover speeds. |
| RPO | < 1 min | Synchronous PostgreSQL streaming replication (primary → standby). |
| Concurrent workflow capacity | 50+ | ~20 provisioning workflows + ~30 drift approval workflows running in parallel at peak. |
| Temporal workflow retention | 90 days active | Completed workflows archived to object storage beyond 90 days. |
| Graceful quota degradation | Platform soft quota enforces limits | If cloud provider quota API is unavailable, the platform soft quota layer continues to block over-provisioning. |
| Drift snooze | Per-MR configurable | Suppresses repeated notifications across scan cycles for known, accepted drift. |

---

## 3. Scale Analysis

```
Tenants:             100
Users:               1,000  (peak concurrent: ~200)
Resources per tenant: 15 avg → 1,500 total XR/MR objects in etcd
Cloud providers:     2 (OneCloud + GCP) × 5 services = 10 resource types
Provisioning load:   20 requests/day, scattered 09:00–17:30

Drift scan load (Approach D, two-phase):
  Phase 1: LIST per GVR across all MRs with drift-protection label
  Phase 2: 10 GVRs × 100 tenants = up to 1,000 ScanTenantActivity tasks per scan cycle
  Schedule: every 1 minute, overlap policy SKIP

Audit log volume:
  ~100 events/day per active tenant = 10,000 events/day
  3 years: ~10.9M rows @ ~300 bytes/row ≈ 3.3 GB
  PostgreSQL with monthly range partitioning handles this without concern.
```

---

## 4. Kubernetes Cluster Topology

### Option A — Single Cluster per Environment

All components run in one multi-AZ Kubernetes cluster per environment.

```
[Production K8s Cluster — multi-AZ]
  ucp-platform ns      API server, Vault, ingress
  temporal-system ns   Temporal Server, Temporal Workers
  crossplane-system ns Crossplane, Providers
  ucp-workers ns       provisioning-worker, drift-worker
  ucp-db ns            PostgreSQL (platform + temporal, separate schemas)
  monitoring ns        MonaaS agent, EaaS log shipper
```

### Option B — Platform + Operations Cluster (two clusters per environment)

Platform cluster holds API-facing components. Operations cluster holds Crossplane and
Temporal.

```
[Platform Cluster — multi-AZ]       [Operations Cluster — multi-AZ]
  API Server ×2                        Temporal Server (HA)
  PostgreSQL Platform (HA)             PostgreSQL Temporal (HA)
  Vault (HA)                           Temporal Workers (provisioning, drift)
  Monitoring (MonaaS + EaaS)           Crossplane (HA)
  Ingress + LoadBalancer               Crossplane Providers (gcp, roc)
                                       External Secrets Operator
                                       KEDA
```

### Option C — Three Clusters (Platform + Crossplane + Temporal)

```
[Platform Cluster]     [Crossplane Cluster]     [Temporal Cluster]
  API server             Crossplane               Temporal Server
  PostgreSQL Platform    Providers                PostgreSQL Temporal
  Vault                  Temporal Workers*
  Monitoring

  * Workers must co-locate with Crossplane for in-cluster K8s API access (ScanDriftActivity).
```

### Provider-per-cluster (excluded)

Splitting Crossplane by cloud provider is architecturally incorrect. Crossplane is
designed as the single multi-cloud control plane — provider plugins reach out to GCP
and OneCloud via their respective APIs from one cluster. Splitting by provider gains
nothing at this scale and breaks the composition model.

---

### Topology Comparison

| Dimension | A — Single | B — Platform + Ops | C — Three Clusters |
|---|---|---|---|
| **Availability** | Crossplane MR churn, Temporal history writes, and API traffic all share one etcd and one K8s API server. Write spikes from drift scans can affect API read latency. | Operations cluster failure stops provisioning and drift — API still serves reads, sessions, and RBAC. Platform cluster failure stops API — Temporal workflows continue until timeout. Blast radius is clear. | Each tier fails independently. Maximum isolation per component. |
| **Blast radius** | Any component failure can cascade to the entire system via shared etcd and shared node pressure. | Crossplane and Temporal issues are contained to the Ops cluster. API stability is independent. | Full isolation. One cluster's instability has zero impact on others. |
| **Scalability** | All components scale together. KEDA-triggered drift worker scale-out competes with API server pods for node resources. | Ops cluster scales independently of Platform. KEDA can aggressively scale drift workers without impacting API pods. | Fine-grained per-tier autoscaling. |
| **etcd isolation** | Shared etcd: Crossplane MR status patches (thousands per minute during drift scans) compete with platform objects and session lookups. Under load, etcd write latency affects all K8s API calls on the cluster. | Separate etcds: Crossplane and Temporal MR churn is confined to Ops cluster etcd. Platform cluster etcd serves stable, low-write objects only. API server read latency is not affected by Crossplane activity. | Fully isolated etcds. |
| **Operational complexity** | One cluster per env. One upgrade cycle. Simple kubeconfig. | Two clusters per env. Cross-cluster kubeconfig (API server → Ops K8s API for XR operations). Two PostgreSQL instances. Cross-cluster Temporal gRPC adds ~2–5ms latency per call — within the 1s API budget. | Three clusters. Multiple cross-cluster connections. Distributed failure diagnosis is significantly harder. |
| **Cost** | Lowest. Shared node pools, one control plane fee. | Medium. Two control plane fees, separate node pools. | Highest. Three control plane fees. |
| **RBAC isolation** | All ServiceAccounts share one K8s API server. A misconfigured Crossplane SA could affect platform objects. | RBAC isolated per cluster. Crossplane SA can only touch Ops cluster resources. | Full per-tier RBAC isolation. |
| **K8s upgrade** | One upgrade window, all components affected simultaneously. Crossplane, Temporal, and API server compatibility must be validated together. | Ops and Platform upgrade independently. API tier upgrade does not require validating Crossplane version constraints. | Fully independent upgrade per tier. |
| **Temporal worker placement** | In-cluster K8s access is free — no extra kubeconfig. | Workers run on Ops cluster. K8s API access is in-cluster (same cluster as Crossplane). Temporal Server on Ops cluster, or on Platform cluster with cross-cluster gRPC. | Workers must stay with Crossplane. Temporal Server is a separate cluster with cross-cluster gRPC from workers. |
| **For MVP (100 tenants)** | Acceptable. | Recommended. | Over-engineered. |
| **For scale (500+ tenants)** | etcd write pressure from drift scans starts affecting API latency. Node resource contention becomes measurable. | Scales well. Each tier grows independently. | Best headroom. |

---

### Recommendation: Option B

Option B is the recommended topology. At 1,500 MRs across 10 GVRs, the drift scan
generates significant K8s API write traffic during each scan cycle. Keeping that on a
separate etcd means the Platform cluster's etcd remains consistently fast for session
lookups, RBAC reads, and audit writes. The blast radius separation between API
availability and operational availability is operationally important — a Crossplane
reconciliation issue or provider outage does not take down the API for end users.

The ~2–5ms cross-cluster overhead for API server calls to Temporal and to the Ops K8s
API is well within the 1s API latency budget.

**Path to Option C:** If at 500+ tenants Temporal history service memory pressure or
Crossplane provider pod resource contention becomes measurable on the Ops cluster,
split Temporal onto its own cluster. Temporal Workers stay on the Crossplane cluster.
This is a targeted migration with no API server changes.

---

## 5. Per-Environment Cluster Layout (Option B)

Two environments: QA and Production. Each environment is a fully independent set of
clusters with the same layout.

```
[Platform Cluster — multi-AZ, 3 AZs]
  Namespace: ucp-platform
    API Server              2 replicas, PodAntiAffinity (one per AZ)
  Namespace: ucp-db
    PostgreSQL Platform     Primary (AZ-1), Sync Standby (AZ-2), Async Standby (AZ-3)
  Namespace: vault-system
    Vault                   3-node Raft cluster, one per AZ
  Namespace: monitoring
    MonaaS agent, EaaS log shipper
  Ingress: nginx LoadBalancer, TLS termination

[Operations Cluster — multi-AZ, 3 AZs]
  Namespace: temporal-system
    Temporal Frontend       2 replicas
    Temporal History        2 replicas
    Temporal Matching       2 replicas
    Temporal Worker Svc     2 replicas
    PostgreSQL Temporal     Primary (AZ-1), Sync Standby (AZ-2), Async Standby (AZ-3)
  Namespace: crossplane-system
    Crossplane              2 replicas, leader election
    provider-upjet-gcp      2 replicas, leader election
    provider-roc            2 replicas, leader election
  Namespace: ucp-workers
    provisioning-worker     2 replicas (fixed)
    drift-worker            2–10 replicas (KEDA managed)
  Namespace: keda-system
    KEDA                    2 replicas
  External Secrets Operator (cluster-scoped)
```

---

## 6. Component Design

### API Server

Two replicas on the Platform cluster, spread across AZs via PodAntiAffinity.

The API server requires two outbound connections to the Operations cluster:

- **Temporal gRPC:** reaches Temporal Frontend via an Internal LoadBalancer on the Ops
  cluster. Used to start and query workflows.
- **Ops K8s API (kubeconfig):** a scoped ServiceAccount on the Ops cluster with
  `get/list/create/delete` on XR and MR GVRs only. Used for XR lifecycle operations
  and for reading status on GET requests.

All other connections (PostgreSQL, Vault, Horizon OIDC) remain on the Platform cluster
or are external endpoints. The cross-cluster calls add ~2–5ms per request, absorbed
within the 1s API budget.

The Clean Architecture design from the backend architecture document applies unchanged.
The infrastructure layer's K8s client and Temporal gateway are the only components that
carry cross-cluster configuration.

---

### PostgreSQL — Two Instances

Two independent PostgreSQL deployments with separate schemas, sizing, and backup
policies.

**Why two instances, not one:**

| Concern | Platform DB | Temporal DB |
|---|---|---|
| Schema ownership | UCP team | Temporal Server version drives schema |
| Access patterns | Low-frequency reads and writes (sessions, RBAC, audit) | High-frequency sequential writes (workflow history, visibility) |
| Retention policy | 3-year audit logs drive storage sizing | 90-day workflow retention |
| Upgrade coupling | Independent | Temporal Server version must match schema version |
| Failure scope | Platform DB down → API requests fail; Temporal continues running | Temporal DB down → provisioning and drift stop; API reads still work |

A Platform DB maintenance window does not halt Temporal. A Temporal schema migration
does not risk the sessions or audit logs tables.

**HA configuration (both instances, single region, multi-AZ):**

```
Primary            (AZ-1)  — handles all writes
Sync Standby       (AZ-2)  — RPO < 1 min, automatic failover via Patroni
Async Read Replica (AZ-3)  — audit log queries and reporting reads
```

Patroni manages leader election and automatic failover. On GCP: Cloud SQL HA with read replica covers this natively with managed ops.

**Audit log partitioning:**

```sql
CREATE TABLE audit_logs (
    id          UUID DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id),
    session_id  TEXT,
    action      TEXT NOT NULL,
    resource    TEXT NOT NULL,
    metadata    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);
```

Monthly partitions: `audit_logs_2026_06`, `audit_logs_2026_07`, and so on. Oldest
partitions are archived to object storage after 12 months and dropped without table lock.
This keeps the active table small while satisfying the 3-year retention requirement.

**Temporal visibility store:** PostgreSQL visibility is sufficient at 100 tenants and
20 provisioning requests per day. If tenant count exceeds 500 and workflow list query
latency becomes measurable, switch the visibility store to Elasticsearch or OpenSearch.
The default store (workflow history) stays in PostgreSQL regardless.

---

### Temporal — Self-Hosted HA

Temporal Server runs as four internal services on the Operations cluster.

| Service | Role | Replicas | State |
|---|---|---|---|
| Frontend | Routes SDK calls, serves Temporal Web UI and API | 2 | Stateless |
| History | Owns and persists workflow state shards | 2 (4 shards) | PostgreSQL-backed |
| Matching | Routes activity and workflow tasks to workers | 2 | Stateless |
| Worker (internal) | Timers, archival, workflow replication | 2 | Internal |

PodAntiAffinity spreads each service across AZs. Deployed via `temporalio/helm-charts`.

**Task queues:**

Two task queues with separate worker deployments:

- `provisioning` — RequestDatabase, RequestCompute, RequestStorage, and related
  workflows. Fixed 2 replicas. Load is predictable at 20 requests per day.
- `drift-detection` — DriftScanWorkflow, DriftApprovalWorkflow, and all associated
  activities. KEDA-managed autoscaling. Load bursts every scan cycle.

Queue separation means a surge in drift scan activity cannot starve provisioning
workflows, and provisioning worker replicas are not over-provisioned to absorb drift
scan bursts.

**Workflow retention:** 90 days active in PostgreSQL. Temporal's archival feature moves
completed workflows to GCS beyond 90 days. DriftScanWorkflow runs every minute — at
90-day retention this accumulates ~129,600 scan workflow entries. Archival prevents
unbounded PostgreSQL growth.

---

### Temporal Workers — Two Deployments

**provisioning-worker (Platform: Operations Cluster)**

```
Replicas: 2 (fixed)
Task queue: provisioning
Registers:
  - RequestDatabaseWorkflow
  - RequestComputeWorkflow
  - RequestStorageWorkflow
  - ApplyYAMLActivity
  - WaitReadyActivity
  - ReadSecretActivity
  - NotifyProvisioningActivity
```

Fixed replicas are sufficient. 20 provisioning requests per day with 20–40 minute
workflow duration means at most 3–4 concurrent provisioning workflows during peak hours.
Two replicas with standard concurrency settings handles this with headroom.

**drift-worker (Platform: Operations Cluster, KEDA autoscaled)**

```
Replicas: 2 (min) to 10 (max), KEDA Temporal queue-depth scaler
Task queue: drift-detection
Registers:
  - DriftScanWorkflow
  - DiscoverMRsActivity
  - ScanTenantActivity
  - DriftApprovalWorkflow
  - FlipManagementPolicyActivity
  - WaitMRReadyActivity
  - NotifyDriftActivity (all channels)
```

Each drift scan dispatches up to `10 GVRs × 100 tenants = 1,000 ScanTenantActivity`
tasks per minute. With 2 workers each handling 10 concurrent activities, the queue
depth spikes every minute then empties between scans. KEDA scales up on queue spike and
back to minimum between scans. Fixed replicas would either over-provision (idle between
scans) or under-provision (scan backlog accumulates and violates the 1-minute detection
NFR).

Workers co-locate with Crossplane on the Ops cluster. `ScanDriftActivity` and
`FlipManagementPolicyActivity` use in-cluster K8s API access — no additional auth or
kubeconfig required.

---

### Crossplane — HA on Operations Cluster

```
Namespace: crossplane-system

crossplane controller    2 replicas, leader election
provider-upjet-gcp       2 replicas, leader election
provider-roc             2 replicas, leader election
```

Leader election means one replica actively reconciles at a time. The second replica is
a hot standby that takes over within seconds on leader failure. This satisfies the 99.9%
availability requirement for Crossplane-managed reconciliation.

**Provider poll interval:** `--poll=1m` for all providers. This is the drift detection
floor — increasing it reduces K8s API and cloud provider API call volume. The 1-minute
scan schedule is matched to this interval.

**ProviderConfig per tenant per project:** Each tenant's GCP credentials map to a
distinct ProviderConfig named `gcp-{tenant-id}-{project-id}-{env}`. Credentials are
stored as K8s secrets in `crossplane-system`, synced from Vault by External Secrets
Operator.

**Namespace-per-tenant (future):** Once all required providers ship namespace-scoped MR
support (tracked in MCUCP-119), XRDs move from `scope: Cluster` to `scope: Namespaced`.
Each tenant gets a dedicated namespace `tenant-{slug}` on the Ops cluster. This does not
change the cluster topology.

---

### Secret Management — Vault + External Secrets Operator

**Why not Kubernetes secrets alone:**

K8s secrets are base64-encoded and unencrypted at rest unless KMS envelope encryption
is explicitly enabled. They have no audit trail per-secret-access, no lease or TTL
management, and no dynamic secret generation. For cloud provider credentials subject to
3-year compliance retention and rotation requirements, K8s secrets alone are
insufficient.

**Stack:**

```
HashiCorp Vault (Platform Cluster, 3-node Raft HA)
  Auth method:    Kubernetes (pods present their ServiceAccount JWT, Vault validates)
  Secret engines:
    kv-v2/ucp/tenants/{tenant-slug}/gcp    GCP service account keys
    kv-v2/ucp/tenants/{tenant-slug}/omnia  Omnia credentials
    kv-v2/ucp/platform/                    Platform secrets (session key, etc.)
  Policies:       per-tenant read-only, platform-admin
  Audit backend:  file → shipped to EaaS for 3-year retention

External Secrets Operator (Operations Cluster)
  VaultProvider   reads from Vault using K8s auth (cross-cluster)
  ExternalSecret  syncs Vault secrets to K8s Secrets in crossplane-system
  Refresh:        1h (credentials change infrequently)
```

**Why Vault over GCP Secret Manager:**

GCP Secret Manager is a good fit for a GCP-only platform. UCP manages both GCP and
OneCloud resources and is designed to add further providers. A provider-agnostic secret
manager means the credential storage layer does not change as new cloud providers are
onboarded. Vault also provides per-secret audit trails natively, which GCP SM does not
provide at the same granularity.

**Rotation flow (zero downtime):**

```
Tenant uploads new GCP SA key via API
  → API server writes to Vault (kv-v2/ucp/tenants/{slug}/gcp)
  → ESO refresh cycle (≤1h) detects changed secret version
  → ESO updates K8s Secret in crossplane-system
  → Crossplane picks up new credential on next reconcile cycle
  → No pod restarts required at any step
```

---

### Notification Service

Notifications are implemented as Temporal activities inside `DriftApprovalWorkflow` and
`RequestDatabaseWorkflow`. There is no separate notification microservice.

**Why not a separate service:**

A dedicated notification service would require its own deployment, availability SLA,
retry logic, and dead-letter handling. Temporal provides all of this for free through
activity retries, timeouts, failure visibility, and workflow history. A separate service
here adds operational surface area without any benefit at this scale.

**Three activities, one per channel:**

```go
NotifySlackActivity(ctx, in NotifyDriftInput) error
NotifyPagerDutyActivity(ctx, in NotifyDriftInput) error
NotifyEmailActivity(ctx, in NotifyDriftInput) error
```

Each activity is a separate Temporal task. A flaky Slack webhook does not block
PagerDuty delivery. Each channel retries independently within its own activity timeout.

**Configuration stored in PostgreSQL (Platform DB):**

```sql
CREATE TABLE notification_configs (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    tenant_rns  TEXT NOT NULL,
    user_id     UUID REFERENCES users(id),  -- NULL = tenant-level config
    channel     TEXT NOT NULL CHECK (channel IN ('slack', 'pagerduty', 'email')),
    enabled     BOOLEAN NOT NULL DEFAULT true,
    config      JSONB NOT NULL,             -- {"webhook_url": "..."} etc.
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_rns, user_id, channel)
);
```

`NotifyDriftActivity` reads the enabled channels for the affected tenant, dispatches
to each. Tenant-admin configures channels via the settings UI or CLI.

**Drift snooze:**

The annotation `platform.io/drift-snoozed-until: "2026-06-16T09:00:00Z"` on an MR
suppresses `DriftApprovalWorkflow` starts for that MR until the timestamp. Set via the
approval UI or CLI when a drift event is acknowledged but not immediately actionable.
`ScanTenantActivity` checks this annotation before calling `ExecuteWorkflow`.

---

### Redis — Not Included in MVP

| Proposed use | Verdict | Reason |
|---|---|---|
| Session cache | Not needed | 1,000 active sessions, indexed PostgreSQL lookup is sub-5ms. Redis adds operational overhead without measurable latency benefit at this scale. |
| Quota data cache | Not needed | Quota Option D (lazy first-fetch + PostgreSQL, background Temporal refresh for active tenants) achieves <5ms quota reads after the first warm-up. At 100 tenants, a one-time <300ms cold-start is acceptable. |
| Rate limiting | Not needed | nginx ingress rate limiting is sufficient for 1,000 users. In-memory token bucket in the API server handles burst shaping at this scale. |
| Hash ring propagation | Not needed at MVP | Hash ring logic is present in code from Day 1 and handles N clusters correctly. At launch with a single Ops cluster, the ring state is static and trivial — API servers load it from Platform DB on startup. Cross-replica propagation is not a problem because the ring does not change until a second cluster is added (Year 3+). At that point, Platform DB polling (30s interval) is sufficient since cluster additions are planned operational events, not real-time changes. Redis is warranted only if propagation latency of polling becomes unacceptable. |

Redis is introduced when either: (1) tenant count exceeds ~500 active tenants, at which
point quota check latency from PostgreSQL starts approaching the 200ms budget — introduce
Redis as the Option E hot tier (Redis → PostgreSQL → cloud provider) as specified in the
quota design document; or (2) a second Ops cluster is added and hash ring propagation
latency across API server replicas becomes a concern. Whichever comes first.

---

### Message Broker — Not Included in MVP

Temporal is the message broker for all UCP async operations. Every provisioning event,
drift detection, notification, and approval flows through Temporal workflows with durable
execution, at-least-once delivery, configurable retry policies, and a full audit trail
in the Temporal UI.

| Proposed use | Current solution | Verdict |
|---|---|---|
| Audit log writes | Synchronous PostgreSQL write in the use case layer | Sufficient. At 10,000 events per day, synchronous writes add ~5ms per request. Fits the API budget. Zero-loss guarantee matches the audit durability NFR. |
| Drift notifications | Temporal `NotifyDriftActivity` | Temporal provides durability and retry. No broker needed. |
| Provisioning events | Temporal workflow | Already handled. |
| External event streaming | Not in MVP scope | Deferred. |

When external systems need to subscribe to UCP events in real time (a billing service
consuming provisioning events, a CMDB consuming resource lifecycle events), introduce
**NATS JetStream**. NATS JetStream is Kubernetes-native, operationally lightweight, and
appropriate for internal event bus workloads at this scale. Kafka is not justified unless
multi-datacenter replication or very high throughput is required, neither of which applies
here.

---

### Monitoring

```
MonaaS           Metrics platform (OneCloud managed).
                 Scrapes: API server, Temporal workers, Crossplane, providers
                 Key metrics:
                   UCP SLI: API p99 latency, drift detection lag, provisioning depth
                   Tenant health: per-tenant quota usage, drift status, active workflows
                   Infra: PostgreSQL replication lag, Vault seal status, etcd latency
                 Alerting:
                   DriftScanWorkflow failure rate > 5% in 5-minute window
                   API error rate > 1% for 5 minutes
                   Temporal task queue backlog > 100 for 5 minutes
                   PostgreSQL replication lag > 30s
                   Vault sealed or leader change

EaaS             Log aggregation (OneCloud managed).
                 Aggregates structured logs from all pods
                 Vault audit log file → EaaS for 3-year retention

Note: No distributed tracing. TraceID is included in API error responses
for correlation with logs in EaaS.
```

---

## 7. Data Flows

### Provisioning Request

```
User (CLI)
  POST /api/v1/databases
  │
  ▼
nginx Ingress (Platform Cluster)
  TLS termination, rate limit
  │
  ▼
API Server
  Auth middleware:  validate session → PostgreSQL Platform (session lookup)
  RBAC middleware:  load tenant roles → PostgreSQL Platform (≤2 rows)
  Quota check:      platform soft quota → PostgreSQL Platform (K8s label count)
  Audit write:      INSERT audit_logs → PostgreSQL Platform (synchronous)
  Start workflow:   gRPC → Temporal Frontend (cross-cluster Internal LB)
  Return:           202 Accepted { workflowId }
  │
  ▼ async — Temporal dispatches to "provisioning" task queue
  │
  ▼
provisioning-worker (Operations Cluster)
  ApplyYAMLActivity
    Creates XR → Ops Cluster K8s API (in-cluster)
    Crossplane reconciles XR, renders MR, calls GCP or OneCloud API
  WaitReadyActivity
    Polls XR status → Ops Cluster K8s API (in-cluster)
    Crossplane Observe() confirms Ready=True
  ReadSecretActivity
    Reads connection secret → Ops Cluster K8s API (in-cluster)
  │
  ▼
Workflow completes
User polls GET /api/v1/databases/{name}
  API Server reads XR status → Ops Cluster K8s API (cross-cluster kubeconfig)
```

### Drift Detection and Approval

```
Crossplane provider-upjet-gcp or provider-roc (Operations Cluster)
  Observe() every ~1 min
  Calls GCP or OneCloud API, writes status.atProvider
  │
  ▼
Ops Cluster etcd (via K8s API server)
  MR.spec.forProvider.settings.tier  = "db-n1-standard-2"  (desired)
  MR.status.atProvider.settings.tier = "db-n1-standard-4"  (actual — DRIFT)

Temporal Schedule (every 1 min, overlap policy SKIP)
  Triggers DriftScanWorkflow
  │
  ▼
DriftScanWorkflow (drift-worker, Operations Cluster)
  │
  Phase 1 — parallel per GVR:
  DiscoverMRsActivity(gvr=cloudsql)
    dc.Resource(gvr).List(label: drift-protection=true)
    Returns [(tenant=coupon-team, mr=my-postgres)]
  │
  Phase 2 — parallel per (GVR, tenant) chunk:
  MR list per (GVR, tenant) is split into chunks of DRIFT_SCAN_BATCH_SIZE (default: 100,
  configurable via env var on drift-worker). Each chunk runs as a separate
  ScanTenantActivity in parallel, preventing whale tenants (e.g. CaaS with 5,800 LBaaS
  MRs) from blocking concurrency slots for the entire scan cycle.

  ScanTenantActivity(cloudsql, coupon-team, [my-postgres])
    isDrifted(mr): forProvider vs atProvider diff → DRIFTED
    ExecuteWorkflow("DriftApprovalWorkflow", mrFields + xrFields)
  │
  Phase 3 — fire and forget:
  DriftApprovalWorkflow starts
    │
    NotifyDriftActivity
      Reads NotificationConfig → PostgreSQL Platform (cross-cluster call)
      POST slack webhook
      POST pagerduty events API
      SMTP email
    │
    WaitForApprovalSignal (24h timeout)
    │
    On Approve:
      FlipManagementPolicyActivity  PATCH MR: ["Create","Observe","Update"]
      WaitMRReadyActivity           polls isDrifted() every 10s (35 min timeout)
      FlipManagementPolicyActivity  PATCH MR: ["Observe"]  (always runs)
    │
    On Reject or Timeout:
      NonRetryableError
      MR stays in Observe mode
      Drift re-detected on next scan cycle unless snoozed
```

---

## 8. Open Decisions

Three deployment decisions remain before implementation starts. They do not change the
architecture but affect operational complexity and managed-service dependency.

| Decision | Options | Consideration |
|---|---|---|
| Managed K8s vs self-managed | GKE Standard (managed control plane + etcd) vs kubeadm | GKE Standard removes etcd ops, node repair, and control plane HA from the platform team's responsibility. Recommended given existing GCP footprint. |
| PostgreSQL HA operator | Cloud SQL HA (managed), Patroni (self-managed), Crunchy Data Postgres Operator (K8s-native) | Cloud SQL is ops-light and natively multi-AZ on GCP. Patroni provides more control. Crunchy PGO is a middle ground. Choice has no architectural impact. |
| Vault deployment | New Vault cluster (self-hosted) vs existing internal Vault instance | If Rakuten already operates a HashiCorp Vault instance internally, add a new mount path for UCP rather than running a dedicated cluster. Check before provisioning new infrastructure. |
