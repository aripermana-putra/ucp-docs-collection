---
title: "Architecture Foundations"
space: UCP
parent_page_id: "../production-design.md"
---

# Architecture Foundations

Pre-design questions answered before drawing the architecture. Every component
decision in the system design traces back to these answers.

---

## 1. What does the system actually do?

### Core user-facing operations

| Operation | Type | Notes |
|---|---|---|
| Provision a cloud resource | Async, long-running | Temporal workflow, 20–40 min for databases |
| Deprovision a cloud resource | Async | Temporal workflow, shorter than provisioning |
| View resource status | Sync read | Polls XR status from K8s API |
| Approve or reject a provisioning request | Human signal | Temporal signal, developer submits → tenant-admin approves |
| Approve or reject drift reconciliation | Human signal | Temporal signal with 24h timeout window |
| View and manage quotas | Sync read/write | Pre-provision gate on every create request |
| Upload and manage cloud credentials | Sync write | Stored in Secret Manager (TBD), synced to Crossplane via ESO |
| Manage team roles and access | Sync read/write | Per-tenant RBAC, 1–2 DB rows per request |
| Register a tenant | Sync write | One-time onboarding, triggers OC role sync |

### Failure modes per operation type

Provisioning workflows are the most complex failure surface. A workflow can
fail at any activity step — YAML apply, wait-ready timeout, read-secret. Temporal
retries activities automatically. If the cloud provider API is down, the workflow
parks and retries; the user sees the workflow stuck in-progress.

Drift detection failure is bounded: if a scan cycle fails, the next cycle
(1 minute later) picks up all drifted resources. No state is lost. The snooze
annotation prevents chatty re-notifications between cycles.

Authentication and RBAC failure is a hard stop — requests are rejected, not
queued. The user retries after the dependency (Horizon, PostgreSQL) recovers.

---

## 2. Who are the users and how do they interact?

### User roles

| Role | What they do | How they interact |
|---|---|---|
| Developer | Provision resources, view status, take lifecycle actions on own resources | CLI (MVP), Web UI, open API |
| Tenant Admin | Register tenant, approve provisioning and drift, manage roles, manage credentials, manage quotas | CLI (MVP), Web UI |
| Platform Admin *(TBC)* | Cross-tenant operations, manage tenants, platform-level configuration | CLI (MVP), Web UI |
| *(Reserved — TBC)* | Additional roles (e.g. Auditor: read-only cross-tenant access for compliance and reporting) may be added as requirements are confirmed | TBC |

### Concurrency and load profile

- **20 provisioning requests per day**, scattered across 09:00–17:30 business hours
- Peak concurrent provisioning workflows: 4–5 at any moment, combined with drift approval, let's say 10 concurrent temporal workflow
- Main load is **post-provisioning operational activity**: drift scan (10,000 MR
  objects every minute), status queries, quota checks
- API requests are predominantly reads (list resources, check status) from
  active users (~300 concurrent at peak)
- Batch operations: drift scan dispatches up to `100 GVRs × 100 tenants = 10,000`
  activity tasks per minute — the largest burst in the system

### Latency expectations

| Operation | Target | Dependency |
|---|---|---|
| API request (read, create, delete) | < 200ms | k8s API, db latency |
| Quota check | < 1s | Cloud platform API |
| Drift detection lag (detect to notification) | < 5 min | Crossplane poll lag, status change from cloud platform |

---

## 3. What are my data boundaries?

### Data UCP owns

| Data | Store | Retention | Notes |
|---|---|---|---|
| User sessions | Platform DB | TBD | 
| Blueprint template | Platform DB | Indefinite | 
| Quota policies | Platform DB | Indefinite |
| Notification config | Platform DB | Indefinite |
| Tenant registry | Platform DB | Indefinite |
| Temporal workflow state | Temporal DB | TBD |
| XR / MR objects (desired + observed state) | Kubernetes etcd (Ops crossplane cluster) | Lifecycle of the resource |
| Managed resource (metadata only — name, ID, tenant, type) | Platform DB | Lifecycle of the resource | Read-optimized copy for fast queries. Full desired state lives in etcd (XR/MR). |
| Blueprint run instance | Platform DB | Lifecycle of the resource | related to managed resource
| Provisioning history (Single resource or blueprint) | Platform DB | TBD (3 years ?) |
| Audit logs | Platform DB | TBD (3 years ?) | Might need partition |
| Security policies | Platform DB | Indefinite | Platform-admin defined. Scoped per provider, resource type, tenant. Definition stored as JSONB (OPA rego, structured condition, etc). |
| Policy violations | Platform DB | Lifecycle of the resource | Written on provisioning and scheduled/on-demand scan. Cleared on resolution or resource deletion. detail as JSONB. |

### Data that lives elsewhere

| Data | Owner | How UCP accesses it |
|---|---|---|
| Tenant information, membership and roles | Keycloak / Core Data | OIDC JWT groups claim (from keycloak) + Core Data API |
| Cloud resource actual state | GCP / Omnia APIs | Crossplane Observe() writes to MR status.atProvider |
| Cloud provider credentials | Secret Manager (TBD) | External Secrets Operator syncs to K8s secrets |
| GCP quota metrics | GCP Cloud Monitoring | Fetched on demand, optionally cached |

### Consistency requirements

Session and RBAC data: strong consistency required. PostgreSQL synchronous
replication (RPO < 1 min). Stale sessions must not grant access.

Audit logs: durable, zero loss. Synchronous write before returning the API
response. Not acceptable to lose an audit event silently.

XR/MR state in etcd: Kubernetes guarantees strong consistency within a cluster
via Raft. No cross-cluster consistency needed — each cluster owns its own state.

Managed resource: Eventual consistency is acceptable. Platform DB holds metadata only for fast reads. Source of truth is etcd.

Quota data: eventually consistent is acceptable. Respective cloud platform API will be used for provisioning guardrails.

---

## 4. What are my integration points?

### Inbound — who calls UCP

| Caller | Protocol | Auth method |
|---|---|---|
| CLI (MVP) | HTTPS REST | Keycloak JWT Bearer token |
| Web UI | HTTPS REST | BFF session cookie |
| External integrations (open API) | HTTPS REST | Keycloak JWT Bearer token |
| API-C (gateway layer) | HTTPS | Passes through, UCP validates auth |

### Outbound — what UCP calls

| Dependency | Protocol | Criticality | What happens if it's down |
|---|---|---|---|
| Keycloak | OIDC + REST | High — new logins fail | New logins fail. Token refresh fails when access token expires (TTL ~5–15 min). Active sessions continue working until their access token TTL is exhausted and cannot be refreshed — no immediate outage. |
| Core Data | REST | Low — no runtime impact | Zero impact on active sessions. UCP roles are read from the JWT `groups` claim, not from Core Data API at request time. Role assignments made in the Core Data portal are not reflected until the user's next token refresh. No per-request Core Data call is made. |
| GCP APIs | HTTPS REST | High for provisioning | Crossplane retries Observe/Create/Update/Delete. In-flight workflows park until API recovers. Quota check fails open or falls back to platform soft quota. |
| ROC service provider REST API (Omnia, Athena, etc) | HTTPS REST | High for provisioning | Same as GCP. |
| Temporal Server | gRPC | High — async ops fail | API returns error on workflow submission. Reads (status) still work. |
| Ops K8s API | HTTPS | High — resource ops fail | API cannot create/list/patch XRs / MRs. Read-only degraded mode. |
| Secret Manager (TBD) | HTTPS | Medium — secrets inaccessible | Crossplane cannot refresh credentials. Existing credentials continue until expiry. |
| PagerDuty / Slack / Email | HTTPS | Low | Temporal retries notification activity. Alert is delayed, not lost. |

### Resilience pattern per dependency

Keycloak is wrapped with circuit breaker + retry + timeout (used at login and
token refresh only — not on the per-request path). Cloud platform APIs (GCP,
Omnia, etc) are wrapped with circuit breaker + retry + timeout via `failsafe-go`.
Core Data is not called on the per-request path — no resilience wrapper needed
for authZ. Temporal and K8s API calls are in-cluster or near-cluster — timeout
only (TBD).

---

## 5. What are my operational boundaries?

### UCP owns and operates

- API server (Go/Echo)
- CLI Binary
- Temporal workers (provisioning-worker, drift-worker)
- Platform DB
- Temporal DB
- Secret Manager (TBD)
- Crossplane + provider plugins (provider-upjet-gcp, provider-roc)
- KEDA
- External Secrets Operator

### Another team operates — UCP is a consumer

| Service | Team | UCP dependency | Notes |
|---|---|---|---|
| CaaS clusters | CaaS team (managed) | Runtime for all UCP workloads | Only if we deploy something to OneCloud
| API-C (Kong gateway) | API-C team | Inbound routing, rate limiting, correlation ID |
| ROC–GCP Dedicated Interconnect | Network team | Network path for GCP provider calls and tenant cross-cloud connectivity |
| Keycloak | Core Data team ? | AuthN |
| Core Data API | Core Data team ? | Tenants, membership and role data |
| OneCloud service SAPIs (DBaaS, VMaaS, STaaS, etc) | Respective service teams | Provisioning targets |
| Public cloud provider APIs | Respective cloud provider | Provisioning targets |

### Independently deployable components

API server, Temporal workers, Crossplane, and Platform Database can each be deployed
and upgraded independently. The only hard coupling is:

- Ideally, Crossplane providers must be co-located with Temporal workers (in-cluster K8s
  API access for drift activities) to minimize latency
- External Secrets Operator (ESO) must run on the same cluster as Crossplane (to sync secrets into the
  crossplane-system namespace)

---

## 6. Non-functional requirements

> **Disclaimer:** The targets and mechanisms below are working assumptions as of this writing. They have not been formally confirmed and are subject to change pending management alignment and infrastructure decisions (deployment target, secret manager selection).

### Availability

| Component | Target | Mechanism |
|---|---|---|
| API server | 99.9% | 2 replicas, PodAntiAffinity across AZs |
| Temporal Server | 99.9% | HA stack (2 replicas per service), PostgreSQL sync replication |
| Crossplane + providers | 99.9% | 2 replicas, leader election |
| Database (both instances) | 99.9% | HA deployment TBD — see RFC-005 |
| Secret Manager | 99.9% | TBD — depends on secret manager selection |

### Durability

| Data | Durability mechanism |
|---|---|
| Audit logs | Synchronous DB write before API response |
| Temporal workflow state | DB sync replication (RPO < 1 min) |
| XR/MR state | Kubernetes etcd Raft (RPO = 0 within cluster) |

### Recovery targets

| Metric | Target |
|---|---|
| RTO | < 15 minutes |
| RPO | < 1 minute (synchronous DB replication) |

### Security

- Authentication: Keycloak OIDC (JWT groups claim for tenant/role data)
- Authorization: per-request RBAC at use case layer, permission bitmask model
- Secrets: Secret Manager (TBD) + ESO — credentials never stored in application config
- Audit: every mutating operation writes an audit log entry synchronously
- Cloud credentials: encrypted at rest in Secret manager, scoped per tenant per provider
- Resource (XR/MR) in Crossplane: Tenant isolation at namespace level in the cluster

---

## 7. Scalability dimensions

### What grows

| Dimension | What it affects |
|---|---|
| Tenant count | Crossplane provider informer cache memory, drift scan activity count, DB row count |
| Resources per tenant | etcd object count, Crossplane reconcile frequency, drift scan time, resource table row count in Platform DB |
| API request rate | API server replicas, PostgreSQL connection pool |
| Resource types per provider | Number of sub-provider pods deployed, CRD count, drift scan GVR list |
| Cloud providers | New provider family + sub-provider pods added per provider, drift scan GVR list, notification routing, credential management scope |

### Initial cluster spec (estimated)

Single cluster for all UCP workloads. Split into Platform + Ops only when
there is a measured reason to (see Section 9 — defer list).

**Node spec: 3 × (4 vCPU, 16GB RAM) across 3 AZs**

provider-upjet-gcp follows the family split model — one pod per GCP service
domain. Only sub-providers for services UCP actively uses are deployed.

| Workload | Pods | Memory estimate | Notes |
|---|---|---|---|
| **Crossplane core** | | | |
| crossplane + rbac-manager | 2 | ~512MB | |
| **GCP provider family (MVP)** | | | |
| upbound-provider-family-gcp | 1 | ~256MB | Parent, bootstraps sub-providers |
| provider-gcp-sql | 1 | ~256–512MB | Cloud SQL |
| provider-gcp-container | 1 | ~512MB–1GB | GKE clusters + node pools |
| provider-gcp-compute | 1 | ~256–512MB | GCE instances |
| provider-gcp-storage | 1 | ~128–256MB | GCS buckets |
| **ROC provider (MVP)** | | | |
| provider-roc | 1 | ~256–512MB | Omnia DBaaS and other ROC services. Custom-built by us. May adopt family split pattern (one sub-provider per ROC service, similar to upjet gcp provider) as service coverage grows. |
| **Crossplane functions** | | | |
| function-* (5 functions) | 5 | ~128–256MB each = ~640MB–1.28GB | patch-and-transform, go-templating, etc. |
| **Temporal Server** | | | |
| Frontend + History + Matching | 3+ | ~2–2.5GB | 2 replicas each for HA |
| **Temporal workers** | | | |
| Provisioning workers (KEDA, 1–2) | 1-2 | ~512MB | |
| Drift workers (KEDA, 1–10) | 1–10 | ~512MB–2.5GB | Scales on queue depth |
| **Platform components** | | | |
| API server | 2 | ~512MB | |
| Platform DB (CloudNativePG) | 2 | ~1–2GB | Only if we host it ourselves. 2 replicas for HA |
| Secret Manager | 3 | ~768MB | Only if we host it ourselves. 3-node HA. |
| KEDA | 1 | ~128–256MB | Autoscales temporal workers on Temporal queue depth |
| ESO | 1 | ~128–256MB | Syncs secrets from Secret Manager to K8s Secrets for Crossplane |
| System overhead | — | ~512MB | kube-system, monitoring agents |
| **Total estimate** | | **~10–14GB** | |

Total cluster capacity: 48GB RAM, 12 vCPUs — comfortable headroom at MVP scale.
GKE Cluster Autoscaler adds nodes automatically under load.

**Adding a new cloud provider** adds one provider family pod (~256MB) plus one
sub-provider pod per resource type used (~256–512MB each). Each provider is
independently deployed, resource-limited, and scaled — no impact on existing
providers.

### First bottleneck as system grows

Crossplane provider informer cache. Each provider pod holds an in-memory cache
of all MR objects it manages. At the target ceiling of ~500 tenants × 100
resources = 50,000 resources, provider memory is estimated at 4–6GB combined —
still within the 3-node spec.

**etcd is not a realistic bottleneck at UCP's scale.** At 50,000 MRs reconciling
every 60s with ~3 writes per reconcile, the estimated write rate is ~2,500
writes/second — far below etcd's capacity. On GKE, etcd runs on Google's managed
control plane and is scaled automatically independent of worker node spec. The
actual concern is write burst contention under high Crossplane reconcile load
causing API server latency spikes — validate this with load testing rather than
relying on theoretical thresholds.

### Scaling strategy

The strategy is **measure first, act second**. Tenant count thresholds below are
indicative only — do not split clusters based on tenant count alone.

**Response order when pressure is observed:**

1. **Provider pod hitting memory limit** → increase `DeploymentRuntimeConfig`
   memory limit for the specific provider under pressure. Each sub-provider is
   independent — bumping one does not affect others.

2. **Pod needs more memory than available on its node** → add more nodes
   (horizontal) or switch to larger nodes (vertical). GKE Cluster Autoscaler
   handles this automatically when pod scheduling pressure is detected. Note:
   adding nodes does not affect etcd on GKE — etcd lives on Google's managed
   control plane, not on worker nodes.

3. **API server latency degrading despite healthy provider pods** → investigate
   root cause before acting. Likely causes: reconcile storm from a buggy provider
   (fix the provider), or genuine etcd write contention (open GCP support ticket —
   etcd is Google's responsibility on GKE). Validate with `apiserver_request_duration_seconds`
   metrics in Cloud Monitoring.

4. **Root cause is confirmed Crossplane write volume saturating the single cluster**
   → only then consider splitting into Platform + Ops clusters. Note: the Platform
   cluster would run ~3–4GB of workloads on 3 × 16GB nodes (~8% utilization) —
   this is wasteful and only justified if there is measured evidence that the split
   resolves the latency issue.

5. **Single Ops cluster cannot handle tenant volume** → cluster sharding by tenant.
   Realistically requires ~800,000+ MRs (8,000+ tenants at 100 resources each)
   before etcd write rate approaches any realistic ceiling. Document for completeness,
   not expected at UCP's foreseeable scale.

**Indicative tenant ranges:**

| Range | Primary action |
|---|---|
| 0–500 tenants | Single cluster. Tune provider memory limits as needed. |
| 500–2,000 tenants | Evaluate provider memory pressure. Add nodes if needed. Split only if API server latency is measurably caused by Crossplane. |
| 2,000+ tenants | Cluster sharding by tenant if single Ops cluster is a confirmed bottleneck. |

---

## 8. Security boundaries

### Where authentication happens

The API server validates every request. For CLI and open API clients: Keycloak
JWT Bearer token validated against JWKS endpoint. For web UI: server-side
session cookie validated against PostgreSQL sessions table. API-C sits in front
but does not perform auth validation — it passes headers through to UCP.

### Where authorization happens

At the use case layer, inside the API server. The `RequirePermission` middleware
reads the UCP service role directly from the Keycloak JWT `groups` claim already
present in the request context — zero additional DB or API calls.

UCP is registered as a service in Core Data. The OC tenant-admin assigns UCP
roles (`developer`, `tenant-admin`, `platform-admin`) to their members via the
Core Data portal. Keycloak includes these as group entries in the JWT:

```
rns:roc:ucp::{tenant-slug}:roles:developer
rns:roc:ucp::{tenant-slug}:roles:tenant-admin
```

The middleware looks up `serviceRoles["ucp"]` from the parsed claims and maps it
to the UCP permission bitmask. No DB lookup. No Core Data API call. RBAC is
permission-based (bitmask), not role-hierarchy — adding new roles does not
require schema changes.

**Staleness:** Role changes in Core Data take effect at the user's next token
refresh (access token TTL, typically 5–15 minutes). This is the accepted
trade-off for zero per-request overhead.

### Secret flow

```
Secret Manager (TBD)
  → ESO (syncs to K8s Secret in crossplane-system namespace)
    → Crossplane ProviderConfig (references the K8s Secret)
      → Sub-provider pod (reads secret, calls cloud API)
```

Secrets never flow through the API server. Credential upload writes to the
Secret Manager directly. Rotation is zero-downtime: update Secret Manager →
ESO refreshes within its sync TTL → Crossplane picks up on next reconcile.

Secret Manager selection is TBD — the flow above holds regardless of which
Secret Manager is chosen, as ESO supports all major backends.

### Token TTL as the unified staleness boundary

UCP accepts a bounded staleness window across all session-related state. The
access token TTL is the single clock that governs this window:

| Scenario | Effect | When it resolves |
|---|---|---|
| User's UCP role is changed in Core Data | Old role still accepted | Next token refresh (within TTL) |
| User logs out (CLI) | Old token still accepted by API server | Token expiry (within TTL) |
| User is removed from a tenant in Core Data | Still has access | Next token refresh (within TTL) |

**Decision:** No JTI blacklist. No per-request Core Data call. Logout for CLI
clients revokes the refresh token at Keycloak (preventing new tokens) and
deletes the local credential file. The current access token remains valid until
expiry.

**Rationale:** UCP already accepts role staleness bounded by token TTL. Applying
the same bound to logout is consistent. For an internal platform, a maximum
staleness window of 10–15 minutes is operationally acceptable. This eliminates
Redis as a runtime dependency and keeps every request fully stateless for Bearer
token clients.

**Access token TTL target:** 10–15 minutes. Configurable in Keycloak per realm.
Short enough to bound the staleness window; long enough to avoid excessive
refresh calls from active CLI users.

**Revisit if:** a security audit or compliance requirement mandates immediate
revocation, at which point a JTI blacklist backed by Redis is the correct
addition.

---

### Blast radius

If the API server is compromised: attacker can read/write tenant resources
within the permission scope of the stolen session. PostgreSQL credentials are
the most sensitive secrets the API server holds.

If a Crossplane provider is compromised: attacker has access to cloud provider
credentials for all tenants that provider serves. Mitigation: namespace-per-tenant
(target architecture) limits blast radius to one tenant's credentials per pod.

If the Secret Manager is compromised: all cloud credentials are exposed. Audit logs
provide forensic trail. Rotation invalidates all exposed credentials.

---

## 9. What can I defer?

### Defer — decision is reversible, no MVP blocker

| Item | Why defer | When to revisit |
|---|---|---|
| Redis (quota cache, session cache) | PostgreSQL reads are fast enough at 100 tenants. Adding Redis adds ops overhead with no measurable benefit. | When quota check latency consistently approaches 200ms (likely at 500+ active tenants) |
| Message broker (NATS/Kafka) | Temporal already provides durable async execution for all current use cases. No external consumer of UCP events in MVP. | When external systems need to subscribe to UCP events in real time |
| Cluster sharding | Not needed until Crossplane provider memory pressure is measured in practice | When provider pod memory exceeds 80% of node allocatable consistently |
| Platform + Ops cluster split | Single cluster is sufficient for MVP. Split adds cross-cluster ops overhead and results in ~8% utilization on the Platform cluster — wasteful without a confirmed need. | Only when API server latency degradation is confirmed to be caused by Crossplane write volume, and vertical/horizontal node scaling has already been exhausted. |
| Multi-region active-active | Temporal OSS does not support cross-region workflow replication. Single region with Lv2 redundancy satisfies the current assumption. | If UCP is classified as an emergency prioritized operation or regulatory requirements mandate multi-region |

### Decide now — decision is hard or expensive to reverse later

| Item | Status | Why decide now |
|---|---|---|
| K8s runtime — CaaS vs GKE | Open (OQ#3) | CRD support is a hard blocker for Crossplane. Must confirm before any infrastructure is provisioned. |
| Secret manager selection | Open (OQ#2) | ESO configuration and credential migration depend on this. Changing it later requires migrating all tenant credentials. |
| PostgreSQL HA deployment model | Proposed (RFC-005) | Cloud SQL vs CloudNativePG — choice affects backup strategy, failover time, and operational model. Low cost to decide early, high cost to migrate a live database later. |
| API-C as the inbound gateway | Open | Affects auth model, rate limiting design, and onboarding process. Locking in now means all client tooling (CLI, open API docs) uses the API-C URL pattern. |
| Cloud provider authentication model | Open (OQ#4) | Affects credential storage, ESO usage, and tenant onboarding flow. WIF (GCP) is the target direction — pending validation. |

## Open Questions

1. **Secret manager** — which secret manager to use? Affects ESO configuration and credential storage model.
2. **Deployment target** — where to deploy UCP? OneCloud? GCP? Hybrid? Determines K8s runtime (CaaS vs GKE), database HA option (RFC-005), and network topology.
3. **Cloud provider authentication** — long-lived service account keys are not allowed by the security team. WIF (Workload Identity Federation) is the target direction for GCP — pending hands-on validation. ROC authentication model TBD separately.
4. **Reliability targets** — SLA (99.9%), RTO, RPO, and IT service redundancy level (Lv2/Lv3) are working assumptions. Must be formally confirmed with management before production deployment.

### Reliability targets (unconfirmed — requires management alignment)

Working assumptions only. Must be formally agreed before production deployment.

| Metric | Current assumption |
|---|---|
| SLA (availability) | 99.9% |
| IT service redundancy level | Lv2 minimum ([CIO Instruction 002453](https://officerakuten.sharepoint.com/sites/RGR/Library/CIO%20Guidelines%20&%20Instructions/%5B002453%5DCIO%20Instruction%E3%80%80Information%20Technology%20Business%20Continuity%20Plan%20(IT-BCP)/%5B002453%5DInstruction%20for%20Information%20Technology%20Business%20Continuity%20Plan%20(IT-BCP).pdf?CID=6072ebe5-fffc-40ee-82de-494b4d3b2f78)), targeting Lv3 by choice |
| RTO | < 15 minutes |
| RPO | < 1 minute |
| Drift detection lag SLA | < 5 minutes from detection to notification |