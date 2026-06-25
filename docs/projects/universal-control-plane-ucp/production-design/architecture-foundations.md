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
| Upload and manage cloud credentials | Sync write | Stored in Vault, synced to Crossplane via ESO |
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
| Platform Admin | Cross-tenant operations, manage tenants, platform-level configuration | CLI (MVP), Web UI |

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
| User sessions | Platform DB | xx days | 
| Blueprint template | Platform DB | Indefinite | 
| Quota policies | Platform DB | Indefinite |
| Notification config | Platform DB | Indefinite |
| Tenant registry | PostgreSQL (Platform DB) | Indefinite |
| Role tenant assignment cache | Platform DB | Updated on login | Might need regular background sync
| Temporal workflow state | Temporal DB | 90 days active, archived to object storage |
| XR / MR objects (desired + observed state) | Kubernetes etcd (Ops crossplane cluster) | Lifecycle of the resource |
| Managed resource (Desired state of single resource) | Platform DB | Lifecycle of the resource | Might need regular sync to etcd |
| Blueprint run instance | Platform DB | Lifecycle of the resource | relation to managed resource
| Provisioning history (Single resource or blueprint) | Platform DB | 3 years ? |
| Audit logs | Platform DB | 3 years ? | Might need partition |
| Security policies | Platform DB | Indefinite | Platform-admin defined. Scoped per provider, resource type, tenant. Definition stored as JSONB (OPA rego, structured condition, etc). |
| Policy violations | Platform DB | Lifecycle of the resource | Written on provisioning and scheduled/on-demand scan. Cleared on resolution or resource deletion. detail as JSONB. |

### Data that lives elsewhere

| Data | Owner | How UCP accesses it |
|---|---|---|
| Tenant membership and OC roles | Horizon / Keycloak | OIDC JWT groups claim + Horizon REST API at login |
| Cloud resource actual state | GCP / Omnia APIs | Crossplane Observe() writes to MR status.atProvider |
| Cloud provider credentials | Vault | External Secrets Operator syncs to K8s secrets |
| GCP quota metrics | GCP Cloud Monitoring | Fetched on demand, optionally cached |

### Consistency requirements

Session and RBAC data: strong consistency required. PostgreSQL synchronous
replication (RPO < 1 min). Stale sessions must not grant access.

Audit logs: durable, zero loss. Synchronous write before returning the API
response. Not acceptable to lose an audit event silently.

XR/MR state in etcd: Kubernetes guarantees strong consistency within a cluster
via Raft. No cross-cluster consistency needed — each cluster owns its own state.

Managed resource: Eventual consistent is acceptable, source of truth is in etcd.

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

- Temporal workers must share the same Temporal Server task queues as the API server
- Crossplane providers must be co-located with Temporal workers (in-cluster K8s
  API access for drift activities)
- ESO must run on the same cluster as Crossplane (to sync secrets into the
  crossplane-system namespace)

---

## 6. Non-functional requirements

### Availability

| Component | Target | Mechanism |
|---|---|---|
| API server | 99.9% | 2 replicas, PodAntiAffinity across AZs |
| Temporal Server | 99.9% | HA stack (2 replicas per service), PostgreSQL sync replication |
| Crossplane + providers | 99.9% | 2 replicas, leader election |
| Database (both instances) | 99.9% | Patroni HA: primary + sync standby + async read replica across 3 AZs |
| Secret Manager | 99.9% | 3-node Raft cluster across 3 AZs |

### Durability

| Data | Durability mechanism |
|---|---|
| Audit logs | Synchronous DB write before API response |
| Temporal workflow state | DB sync replication (RPO < 1 min) |
| XR/MR state | Kubernetes etcd Raft (RPO = 0 within cluster) |
| Vault secrets | Raft consensus (RPO = 0 within cluster) |

### Recovery targets

| Metric | Target |
|---|---|
| RTO | < 15 minutes |
| RPO | < 1 minute (synchronous DB replication) |

### Security

- Authentication: Keycloak OIDC (JWT groups claim for tenant/role data)
- Authorization: per-request RBAC at use case layer, permission bitmask model
- Secrets: Vault + ESO — credentials never stored in application config
- Audit: every mutating operation writes an audit log entry synchronously
- Cloud credentials: encrypted at rest in Vault, scoped per tenant per provider
- Traffic on ROC–GCP dedicated interconnect: not encrypted by default — application-level TLS required for sensitive data in transit

---

## 7. Scalability dimensions

### What grows

| Dimension | What it affects |
|---|---|
| Tenant count | Crossplane provider informer cache memory, drift scan activity count, DB row count |
| Resources per tenant | etcd object count, Crossplane reconcile frequency, drift scan time, resource table row count in platform DB |
| API request rate | API server replicas, PostgreSQL connection pool |
| Cloud providers | Provider plugin deployments, drift scan GVR list, notification routing |

### First bottleneck as system grows

Crossplane provider informer cache. Each provider holds an in-memory cache of
all MR objects it manages. At ~500+ tenants (7,500+ resources), provider pod
memory starts approaching node limits. Mitigation: vertical scaling of provider
pods first, then cluster sharding if needed.

### Scaling strategy

- **0–500 tenants**: single Ops cluster, KEDA autoscaling for drift workers,
  vertical pod scaling for providers
- **500–2,000 tenants**: evaluate provider pod memory pressure; split Ops
  cluster from Platform cluster if Crossplane write churn affects API latency
- **2,000+ tenants**: cluster sharding by tenant, shard router in API server,
  per-shard Temporal task queues

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
Vault (Platform Cluster)
  → External Secrets Operator (Ops Cluster)
    → K8s Secret (crossplane-system namespace)
      → Crossplane ProviderConfig (references the secret)
        → Provider plugin (reads secret, calls cloud API)
```

Secrets never flow through the API server. Credential upload writes to Vault
directly. Rotation is zero-downtime: update Vault → ESO refreshes within TTL →
Crossplane picks up on next reconcile.

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

If Vault is compromised: all cloud credentials are exposed. Vault audit logs
provide forensic trail. Rotation invalidates all exposed credentials.

---

## 9. What can I defer?

### Defer — decision is reversible, no MVP blocker

| Item | Why defer | When to revisit |
|---|---|---|
| Redis (quota cache, session cache) | PostgreSQL reads are fast enough at 100 tenants. Adding Redis adds ops overhead with no measurable benefit. | When quota check latency consistently approaches 200ms (likely at 500+ active tenants) |
| Message broker (NATS/Kafka) | Temporal already provides durable async execution for all current use cases. No external consumer of UCP events in MVP. | When external systems need to subscribe to UCP events in real time |
| Cluster sharding | Not needed until Crossplane provider memory pressure is measured in practice | When provider pod memory exceeds 80% of node allocatable consistently |
| Platform + Ops cluster split | Single cluster is sufficient for MVP. Split adds cross-cluster ops overhead without measurable benefit at 100 tenants. | When Crossplane MR write churn starts showing correlation with API server read latency |
| Multi-region active-active | Temporal OSS does not support cross-region workflow replication. Single region with Lv2 redundancy satisfies the current assumption. | If UCP is classified as an emergency prioritized operation or regulatory requirements mandate multi-region |

### Decide now — decision is hard or expensive to reverse later

| Item | Why decide now |
|---|---|
| CaaS vs GKE for Ops cluster | CRD support is a hard blocker for Crossplane. Must confirm before any infrastructure is provisioned. |
| Vault vs delegated secret manager | Secret schema and ESO configuration depend on this. Changing it later requires migrating all tenant credentials. |
| Namespace-per-tenant vs cluster-scoped | Blocked on provider support (MCUCP-119), but the XRD schema decision affects all existing resources. Decide the target and track provider readiness. |
| PostgreSQL HA operator | Cloud SQL vs Patroni vs Crunchy PGO — choice affects backup strategy, failover time, and operational model. Low cost to decide early, high cost to migrate a live database later. |
| API-C as the inbound gateway | Affects auth model, rate limiting design, and onboarding process. Locking in now means all client tooling (CLI, open API docs) uses the API-C URL pattern. |

## Open Questions

1. Database to use ?
2. Secret manager to use ?
3. Where to deploy UCP ? OneCloud ? GCP ? Multiple UCP instance in each supported cloud platform (to cut latency and connectivity configuration (ACL etc)) ?
4. UCP to cloud provider authentication ? Long lived token from service account is not allowed by security team

### Reliability targets (unconfirmed — requires management alignment)

Working assumptions only. Must be formally agreed before production deployment.

| Metric | Current assumption | Status |
|---|---|---|
| **SLA (availability)** | 99.9% | Assumed |
| **IT service redundancy level** | Lv2 minimum (CIO Instruction `[002453]`), targeting Lv3 by choice | Classification TBC |
| **RTO** | < 15 minutes | Assumed |
| **RPO** | < 1 minute | Assumed |
| **Drift detection lag SLA** | < 5 minutes from detection to notification | Assumed |