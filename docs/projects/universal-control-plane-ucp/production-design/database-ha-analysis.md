---
title: "Database HA Deployment Analysis"
space: UCP
parent_page_id: "../production-design.md"
---

# Database HA Deployment Analysis — Platform DB

Detailed analysis of three deployment options for the Platform DB, supporting the
decision that will follow ADR-002 (database engine). If Option C is chosen, ADR-002
is rejected and the engine changes from PostgreSQL to MySQL.

> **Note on read replicas:** Read replica coverage in each option is included
> for completeness. Given UCP's load profile (~300 concurrent users peak, simple
> point lookups, ~0.1 writes/second), read replicas are not justified at MVP
> scale. This section documents the capability for future reference, not as a
> current requirement.

**Working assumptions:**

- IT service redundancy level: **Lv2 minimum** (CIO Instruction `[002453]`) —
  multi-cluster, same DC. Lv3 (multi-AZ, same region) is targeted by design choice,
  not mandate.
- SLA target: **99.9%**
- RTO: **< 15 minutes** (assumed)
- RPO: **< 1 minute** (assumed, synchronous replication)

These targets are unconfirmed. See Open Questions in architecture-foundations.md.

---

## Option A — Cloud SQL (GCP, managed PostgreSQL)

### Architecture

Fully managed PostgreSQL on GCP. Google operates the control plane: patching,
failover, backups, monitoring infrastructure. Two availability configurations:

```
Single zone (ZONAL)
  Primary only — no standby
  Zone failure → instance goes down
  = Lv1 — dev/QA only

Multiple zones (REGIONAL / Highly Available)
  Primary (zone A) ──sync replication──► Standby (zone B)
  Automatic failover ~60s, same region
  = Lv3 — production target
```

The standby does not serve reads. It is a failover target only.

### Read replicas

Separate instances created explicitly via GCP Console or Terraform.
Cloud SQL handles WAL streaming replication setup automatically — no manual
configuration required.

```
Primary
  ├──async WAL──► Read Replica 1 (same region, any zone)
  ├──async WAL──► Read Replica 2 (same region, any zone)
  └──async WAL──► Read Replica 3 (cross-region, e.g. asia-northeast2 Osaka)
```

Each replica is independent — all stream directly from the primary, not
from each other. Initial sync: Cloud SQL takes a snapshot of the primary,
restores to the new replica, then streams WAL to catch up. Time proportional
to database size.

Replication lag within same region (cross-zone): negligible (<1ms network
overhead between zones). Cross-region replica: seconds to tens of seconds
depending on write load and physical distance.

Multiple read replicas are supported. Any replica can be promoted to a
standalone instance at any time.

### HA — replication mechanism

Regional (multiple zones) uses **synchronous replication** between primary and
standby. Primary acknowledges writes only after standby confirms receipt.
RPO = 0. No data loss on failover.

### Failover mechanism

Google manages failover automatically. On primary failure, the standby is
promoted. The **connection endpoint (private IP) does not change** — Cloud SQL
updates its internal routing. Applications reconnect to the same address and
reach the new primary. No HAProxy or connection proxy required.

### Backup and PITR

| Feature | Enterprise | Enterprise Plus |
|---|---|---|
| Automated backups | Daily, configurable window | Daily, configurable window |
| Backup storage | Multi-region (configurable) | Multi-region (configurable) |
| PITR retention | 7 days | 35 days |
| PITR restore | Self-service, via GCP Console or gcloud | Self-service |
| Restore time | Minutes (GCP-managed) | Minutes |

PITR is continuous — not snapshot-based. Restore to any second within the
retention window. Self-service: no ticket to file, no external team dependency.

### Monitoring

Cloud SQL exposes metrics natively to **GCP Cloud Monitoring**. Key metrics
available out of the box:

- `cloudsql.googleapis.com/database/cpu/utilization`
- `cloudsql.googleapis.com/database/memory/utilization`
- `cloudsql.googleapis.com/database/disk/utilization`
- `cloudsql.googleapis.com/database/replication/replica_lag`
- `cloudsql.googleapis.com/database/postgresql/num_backends`
- `cloudsql.googleapis.com/database/postgresql/deadlock_count`

UCP uses **MonaaS** for metrics. Run **postgres_exporter** in the same GCP
network as Cloud SQL — it connects as a regular PostgreSQL client, queries
internal statistics views, and exposes a `/metrics` endpoint. MonaaS scrapes
that endpoint. No installation on Cloud SQL itself is required. Needs
validation in practice.

### SLA

| Edition | SLA | Maintenance downtime |
|---|---|---|
| Enterprise | 99.95% | Up to 30 seconds |
| Enterprise Plus | 99.99% | Sub-second |

Enterprise is sufficient for UCP's 99.9% target.

### Key properties

| Dimension | Detail |
|---|---|
| Engine | PostgreSQL — ADR-002 stands |
| Lv2 satisfied? | Yes (ZONAL = Lv2, REGIONAL = Lv3) |
| Failover | Automatic, ~60s, stable endpoint |
| RPO | 0 (synchronous replication) |
| Ops overhead | Minimal — fully managed |
| PITR | 7 days (Enterprise), self-service restore |
| Monitoring | GCP Cloud Monitoring (MonaaS federation TBC) |
| Network | Cross-interconnect if Platform cluster on OneCloud (~3–8ms per query) |
| Cost | GCP billing |
| Internal precedent | Strong — Flat35, GDSP, RMagazine, Coupon all use Cloud SQL in production |

### References

- [Cloud SQL for PostgreSQL overview](https://cloud.google.com/sql/docs/postgres/introduction)
- [Cloud SQL HA documentation](https://cloud.google.com/sql/docs/postgres/high-availability)
- [Cloud SQL read replicas](https://cloud.google.com/sql/docs/postgres/replication/create-replica)
- [Cloud SQL SLA](https://cloud.google.com/sql/sla)
- Internal: Flat35 (Point Service) Cloud SQL production config — `pageId=6669893419`

---

## Option B — Self-managed PostgreSQL

PostgreSQL managed by **CloudNativePG** (on K8s) or **Patroni** (on VMs/bare
metal). The deployment target determines the compute cost, storage type, and
ops burden — the database architecture and replication mechanism are the same
regardless.

### Deployment options

| Target | Status | Notes |
|---|---|---|
| **VMaaS** | Restricted — currently available to TAM and CPSD service providers only. Exception may be possible for UCP. | VM-based, Patroni + etcd |
| **BMaaS** | Available | Bare metal, Patroni + etcd or external K8s DCS. See BMaaS section below. |
| **CaaS cluster** | Available | K8s pods, CloudNativePG, K8s API as DCS |
| **GKE cluster** | Available (if UCP on GCP) | K8s pods, CloudNativePG, K8s API as DCS, pd-ssd storage |

K8s targets (CaaS, GKE) are simpler to operate — CloudNativePG removes the
need for Patroni and separate DCS infrastructure. VM/bare metal targets
(VMaaS, BMaaS) require Patroni and etcd.

### Architecture — K8s targets (CaaS, GKE)

On Kubernetes, PostgreSQL is managed by **CloudNativePG** — a K8s-native
operator. No Patroni. No separate HA tooling. CloudNativePG is a CNCF sandbox
project built specifically for K8s.

You declare a `Cluster` resource and the operator handles everything:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: ucp-platform-db
spec:
  instances: 2          # primary + 1 standby
  storage:
    size: 50Gi
```

CloudNativePG creates the StatefulSet, PVCs, RBAC, and two Services:
- `ucp-platform-db-rw` → primary only (writes)
- `ucp-platform-db-ro` → replicas only (reads)

Failover is automatic — CloudNativePG detects primary failure via K8s liveness
probes and promotes the standby. The `-rw` Service updates within seconds.
No HAProxy, no separate routing component.

---

### Architecture — VM/Bare Metal targets (VMaaS, BMaaS)

On VMs or bare metal, PostgreSQL is installed directly on the server.
**Patroni** manages HA — it is a Python daemon that runs alongside PostgreSQL
on each node and coordinates leader election via a **Distributed Configuration
Store (DCS)**.

On VMs/bare metal there is no K8s API available, so a dedicated DCS must be
deployed separately. The most common choice is **etcd**, which requires a
minimum of 3 nodes for quorum:

```
Server 1 (DC1):  PostgreSQL + Patroni + etcd node 1   (primary)
Server 2 (DC2):  PostgreSQL + Patroni + etcd node 2   (standby)
Server 3 (DC3):  etcd node 3 only                     (tiebreaker)
```

The tiebreaker node (Server 3) exists solely to give etcd quorum — if DC1
goes down, DC2 and DC3 still have majority (2 of 3) and Patroni can elect
a new primary. Without it, losing one server leaves etcd with no majority
and the database cluster freezes.

Failover routing on VMs also requires **HAProxy** as a gateway layer — it
polls Patroni's REST API to track which node is the current primary and
routes writes there. This is the stable endpoint the application connects to.

**Minimum VM/bare metal infrastructure for HA:**

| Server | Runs | AZ |
|---|---|---|
| Server 1 | PostgreSQL + Patroni + etcd | AZ-A |
| Server 2 | PostgreSQL + Patroni + etcd | AZ-B |
| Server 3 | etcd tiebreaker only | AZ-C |
| HAProxy | Routing layer | Any |

3 servers minimum across 3 different AZs for proper redundancy.

---

### HA — replication mechanism (both paths)

PostgreSQL native **WAL streaming replication**. Every write goes to the WAL
first and is streamed continuously to standby nodes. Synchronous mode: primary
waits for at least one standby to confirm receipt before acknowledging the
write — RPO = 0.

### Read replicas (both paths)

All standbys serve reads and act as failover candidates simultaneously. On
K8s: the `-ro` Service routes to standby pods. On VMs: HAProxy routes reads
to standby nodes.

### Backup and PITR

No managed backup — you own the pipeline. Approach differs by deployment target:

**K8s targets (CaaS, GKE):** CloudNativePG provides a `ScheduledBackup` CRD
that triggers backups to an object store (STaaS or GCS). PITR is supported via
WAL archiving. You configure the backup schedule, storage target, and retention.
Restore is self-service via CloudNativePG but manual — you run the recovery procedure.

**VM/bare metal targets (VMaaS, BMaaS):** Use **pgBackRest** — the industry
standard for PostgreSQL backup. Supports full, incremental, and differential
backups with PITR via WAL archiving. Backup storage (STaaS or GCS), retention,
and restore are entirely your responsibility.

### Storage consideration on CaaS

CaaS supports NFS-backed and STaaS persistent volumes. Network-attached storage
adds I/O latency compared to local SSD. For UCP's load profile (~0.1
writes/second, simple queries), NFS latency is unlikely to be noticeable in
practice. See the trade-off section at the end of this document for the broader
debate on databases on Kubernetes.

### Monitoring

**K8s targets:** postgres_exporter + CloudNativePG `/metrics` endpoint → MonaaS.
Both run in the same cluster as the database pods.

**VM/bare metal targets:** postgres_exporter + node_exporter → MonaaS.
postgres_exporter runs on each server, same network as the database.

### SLA

No SLA commitment — availability is as good as you operate it. The underlying
infrastructure (CaaS, GKE, BMaaS) has its own SLA, but database-layer
availability is your responsibility.

### Key properties

Properties below reflect the K8s deployment path (CaaS or GKE with
CloudNativePG), which is the recommended path. VM/bare metal adds Patroni,
etcd, and HAProxy to the ops burden.

| Dimension | Detail |
|---|---|
| Engine | PostgreSQL — ADR-002 stands |
| Lv2 satisfied? | Yes — 2 pods/nodes in same DC |
| Failover | K8s: Automatic via CloudNativePG, ~30s. VM/bare metal: Automatic via Patroni, ~30–60s + HAProxy |
| RPO | 0 (synchronous replication) |
| Ops overhead | K8s: Medium — CloudNativePG handles HA and failover; backup and DB upgrades are yours. VM/bare metal: High — Patroni, etcd, HAProxy, OS patches, backup pipeline all manual. |
| PITR | K8s: CloudNativePG `ScheduledBackup` CRD, storage target configured by you. VM: pgBackRest, fully self-managed. |
| Monitoring | postgres_exporter + CloudNativePG metrics → MonaaS (K8s) |
| Network | Local to deploy target — OneCloud (CaaS/BMaaS) or GCP (GKE) |
| Storage | NFS PV (CaaS) / pd-ssd (GKE) / local disk (BMaaS) |
| Cost | ~¥20,928/month CaaS shared nodes. ~¥0 additional on existing GKE cluster. ~¥63,246/month BMaaS (3 nodes, HA). |
| Internal precedent | None for self-managed PostgreSQL on any OneCloud target |

### GKE sub-option

If UCP's platform cluster is deployed on GCP (GKE), PostgreSQL managed by
CloudNativePG can run on the same GKE cluster instead of Cloud SQL. The
architecture is identical to CaaS — same operator, same K8s Service for
stable endpoint. The key difference is storage.

**Storage advantage over CaaS:** GKE supports Persistent Disk SSD (pd-ssd),
which is block storage with low, consistent I/O latency. No NFS overhead.
Comparable to bare metal for database workloads.

**Cost:** PostgreSQL pods run on the **same GKE cluster** as UCP's other
workloads. GKE charges per node, not per pod. If existing nodes have spare
capacity, the incremental cost of adding PostgreSQL pods is **marginal — near
zero**. A new node is only needed if the cluster is already at capacity.

**Trade-off vs Cloud SQL:** Cloud SQL costs ~¥17,000/month always. PostgreSQL
on a shared GKE cluster costs near zero additionally if the cluster is already
provisioned. Same region, same network, no cross-cloud overhead. The difference
is ops burden: Cloud SQL is fully managed, PostgreSQL on GKE is yours to
operate. If the cluster already exists, the cost argument strongly favors
self-managed PostgreSQL on GKE over Cloud SQL.

| | CaaS + CloudNativePG | GKE + CloudNativePG | Cloud SQL |
|---|---|---|---|
| Storage | NFS (network I/O) | pd-ssd (block I/O) | GCP SSD (managed) |
| Cost | ~¥20,928 shared | ~¥0 additional on existing cluster | ~¥17,000 |
| Managed? | No | No | Yes |
| Location | OneCloud | GCP | GCP |

GKE + CloudNativePG is the self-managed PostgreSQL option for teams on GCP
who want the engine control without the NFS storage penalty of CaaS.

### BMaaS reference

If CaaS is unavailable and a VMaaS exception cannot be obtained, BMaaS is the
fallback. PostgreSQL + Patroni on bare metal. Architecture and replication are
identical — only the compute unit changes. Requires either co-located etcd (3
nodes minimum across different AZs) or CaaS/GKE K8s API as the DCS.

| | CaaS (pods) | GKE (pods) | BMaaS (bare metal) |
|---|---|---|---|
| Monthly cost | ~¥20,928 shared | ~¥0 additional on existing cluster | ~¥63,246 (3 nodes, HA) |
| Storage | NFS PV | pd-ssd | Local disk (best I/O) |
| Right-sized for UCP? | Yes | Yes | No — overkill |
| Stable endpoint | K8s Service | K8s Service | HAProxy or CaaS DCS |
| DCS | K8s API (built-in) | K8s API (built-in) | etcd (3 nodes) or external K8s |

BMaaS is only justified if local disk I/O is a hard requirement, which it is
not at UCP's scale.

### References

- [Patroni documentation](https://patroni.readthedocs.io/)
- [CloudNativePG operator](https://cloudnative-pg.io/)
- [Zalando Postgres Operator](https://github.com/zalando/postgres-operator)
- [pgBackRest documentation](https://pgbackrest.org/)
- [postgres_exporter](https://github.com/prometheus-community/postgres_exporter)
- [CaaS pricing](https://onecloud.rakuten-it.com/one-docs/docs/Compute/CaaS/caas-pricing)
- [BMaaS pricing](https://onecloud.rakuten-it.com/one-docs/docs/Compute/BMaaS/bmaas-pricing)
- [GKE Persistent Disk pricing](https://cloud.google.com/compute/disks-image-pricing)
- [Deploy PostgreSQL to GKE using CloudNativePG](https://docs.cloud.google.com/kubernetes-engine/docs/tutorials/stateful-workloads/cloudnativepg)

---

## Option C — MySQL on OneCloud DBaaS

### Architecture

Managed MySQL service operated by the DBaaS team. Uses VMaaS VMs under the
hood with GTID-based semi-synchronous replication. The service is distinct from
the legacy OneCloud DBaaS (MariaDB/MySQL 5.7) — it is a newer offering called
"MySQL Primary-Replica" with different architecture and SLA.

```
Primary (read + write)
  ├──semi-sync WAL──► Replica 1 (read only)
  ├──semi-sync WAL──► Replica 2 (read only)
  └──one-way async──► Standby DC (read-only backup, Lv3 path)
```

The primary handles all writes and reads. Replicas handle read-only traffic.
All nodes are in a single DC.

### Read replicas

Multiple read replicas are supported within the single DC with automated
scale-out. Replicas can be added or removed via the DBaaS portal.

### HA — replication mechanism

**Semi-synchronous replication (GTID-based)**. The primary waits for at least
one replica to acknowledge receipt before committing. Zero data loss during node
failures. This is stronger than standard async MySQL replication.

### Failover mechanism

**MySQL Primary Replica does not support high availability or automatic
failover.** This is confirmed by the official DBaaS FAQ:

> *"No, it does not support high availability solutions. DBaaS involvement is
> required in case of primary server is down."*
> — [OneCloud DBaaS FAQs](https://onecloud.rakuten-it.com/one-docs/docs/Database/DBaaS/dbaas-faqs)

**Within DC (host failure only):** vMotion can automatically migrate the MySQL
VM to another host in the same DC if the physical host fails. Same IP retained.
This is not HA in the database sense — it is VM-level host recovery.

**Primary failure (any other cause):** If the MySQL primary goes down for any
reason other than a simple host failure that vMotion can handle, the replica
does NOT automatically promote. The DBaaS team must manually promote the replica
to primary. RTO is unknown — depends on DBaaS team response time.

### Backup and PITR

| Feature | Detail |
|---|---|
| Automatic backups | Yes — managed by DBaaS team |
| Binlog backups | Half-hourly |
| PITR granularity | ~30 minutes |
| Backup retention | 7 days |
| Backup storage | STaaS (migrated from Hadoop in v3.0.0, May 2026) |
| Restore | **Professional service** — must file a request with DBaaS team |

PITR is technically supported via binlog backups but restore requires DBaaS team
involvement. Self-service restore is not available. For UCP's 3-year audit log
retention requirement, audit data must be archived to cold storage (STaaS/GCS)
independently — neither the 7-day backup window nor a self-serve restore path
covers this.

### Monitoring

**MonaaS (Monitoring as a Service)** — OneCloud's managed monitoring platform.
Provides database performance metrics out of the box, managed by the DBaaS team.

MonaaS is UCP's metrics platform. Since DBaaS MySQL also uses MonaaS natively,
this is the best-aligned option for metrics — no additional integration work
required.

### SLA

| Topology | SLA |
|---|---|
| Single DC (Primary-Replica) | **99.95%** |
| Multi-DC | **99.99%** |

99.95% (single DC) exceeds UCP's 99.9% target. Multi-DC (99.99%) would require
the cross-DC offering, which moves to Lv3.

### Engine note

MySQL 8.0.x / 8.4.x — not PostgreSQL. If this option is chosen, ADR-002
(PostgreSQL engine) is rejected and a new ADR must be written recording the
engine change and its rationale (OneCloud has no managed PostgreSQL; MySQL is
the managed alternative).

### Key properties

| Dimension | Detail |
|---|---|
| Engine | **MySQL** — ADR-002 rejected, new ADR required |
| Lv2 satisfied? | Yes — Primary-Replica, single DC |
| Failover | **No automatic failover.** vMotion handles host-level failure within DC only. Any primary failure requires manual DBaaS team intervention. RTO unknown. |
| RPO | 0 (semi-synchronous replication) |
| Ops overhead | Minimal — DBaaS team manages |
| PITR | 30-min granularity, 7-day retention, **restore requires DBaaS team** |
| Monitoring | MonaaS native |
| Network | Local to OneCloud — no cross-cloud dependency |
| Cost | Internal billing (lowest) |
| Internal precedent | Strong — many Rakuten teams use OneCloud DBaaS |

### References

- [OneCloud MySQL Primary-Replica intro](https://onecloud.rakuten-it.com/one-docs/docs/Database/DBaaS/mysql-primary-replica/singledc/mysql-primary-replica-intro)
- [DBaaS MySQL Primary-Replica v3.0.0 release notes](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6597402687)
- [DBaaS manual restore operation](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6434577007)
- [OneCloud DBaaS FAQs — HA and failover](https://onecloud.rakuten-it.com/one-docs/docs/Database/DBaaS/dbaas-faqs)

---

## Comparison Summary

| Dimension | A — Cloud SQL | B — Self-managed PostgreSQL | C — DBaaS MySQL |
|---|---|---|---|
| **Engine** | PostgreSQL ✓ | PostgreSQL ✓ | MySQL — ADR-002 rejected |
| **Lv2 satisfied** | Yes (ZONAL) | Yes | Yes |
| **SLA** | 99.95% (Enterprise) | No commitment | 99.95% (single DC) |
| **Replication** | Synchronous (HA standby) | Synchronous (configurable) | Semi-synchronous |
| **RPO** | 0 | 0 | 0 |
| **Failover** | Automatic, ~60s, stable endpoint | Automatic via CloudNativePG, ~30s | **No automatic failover** — manual DBaaS intervention on primary failure |
| **Read replicas** | Yes, separate instances, async | Yes, all standbys serve reads | Yes, multiple within DC |
| **PITR** | 7 days, continuous, self-service | Manual pipeline (pgBackRest), self-managed | 30-min granularity, 7 days, **DBaaS team required** |
| **Ops overhead** | Minimal | K8s: Medium. VM/bare metal: High | Minimal |
| **Storage** | GCP SSD (managed) | NFS PV (CaaS) / pd-ssd (GKE) / local disk (BMaaS) | DBaaS managed |
| **Cost (Platform DB only)** | ~¥17,000/month | ~¥20,928 CaaS shared / ~¥0 additional on existing GKE cluster | ~¥12,650/month |
| **Monitoring** | postgres_exporter → MonaaS (GCP network) | postgres_exporter → MonaaS | MonaaS native |
| **Network dependency** | Cross-interconnect if app on OneCloud | Local to deploy target | Local to OneCloud |
| **Billing** | GCP | OneCloud internal (CaaS/BMaaS) / GCP (GKE) | OneCloud internal |
| **Internal precedent** | Strong (multiple teams) | None | Strong (multiple teams) |

---

## Running Databases on Kubernetes — Community Perspective

Option B deploys PostgreSQL inside a K8s cluster (CaaS). This is a topic with
a strong community opinion history worth understanding before committing.

### Historical stance (2018–2020): don't do it

The dominant view was to keep databases off Kubernetes. StatefulSets were
immature, persistent volume handling was unreliable, operators barely existed,
and every K8s upgrade was a potential disruption to the data layer.

### Current stance (2024–2026): it depends

The consensus has shifted. Mature operators (CloudNativePG, Zalando Postgres
Operator, Crunchy PGO, Percona Operator) have significantly reduced the
operational gap versus bare metal.

**Arguments for databases on K8s:**

- Unified platform — one tool, one monitoring stack, one deployment model for
  teams already running K8s.
- K8s handles operational primitives — automatic pod restart, rescheduling on
  node failure, resource limits.
- Operator maturity — production-grade PostgreSQL operators handle HA, failover,
  upgrades, and scaling. The manual work has shrunk considerably.
- Cost efficiency — if you're already paying for a dedicated cluster, the DB
  runs there at marginal cost.

**Arguments against:**

- **Storage is the hardest problem** — the most cited concern. Network-attached
  PVs (NFS, cloud disks) add I/O latency that bare metal never has. Local
  storage gives better performance but removes pod mobility — if the node dies,
  the pod can't reschedule without data loss unless storage is replicated.
- **Performance ceiling** — bare metal always wins on raw I/O. Container
  networking and cgroup overhead are measurable for write-heavy workloads.
- **K8s upgrade risk** — cluster upgrades can disrupt StatefulSet pods.
  Database upgrades and K8s upgrades become entangled.
- **Expertise required** — PersistentVolumeClaims, StorageClasses, StatefulSets,
  PodDisruptionBudgets. Getting this wrong means data loss.

**Google's recommendation:**
> *Managed services are the recommended default. Kubernetes databases are
> reserved for specific use cases where control and customization justify
> the operational complexity.*

### For UCP specifically

UCP's database workload is not I/O intensive — ~0.1 writes/second, simple
point lookups, ~300 concurrent users peak. The performance argument against
K8s is irrelevant at this scale. NFS-backed PVs on CaaS will not be a
bottleneck. Running PostgreSQL on K8s with CloudNativePG is a reasonable
production choice.

**References:**
- [Google Cloud: To run or not to run a database on Kubernetes](https://cloud.google.com/blog/products/databases/to-run-or-not-to-run-a-database-on-kubernetes-what-to-consider)
- [CockroachDB: Running databases on Kubernetes](https://www.cockroachlabs.com/blog/kubernetes-databases/)

---

## Recommendation

### 1st — Cloud SQL (Option A)

**When:** UCP platform is deployed on GCP, and cost is not the primary concern.

Cloud SQL is the first recommendation because UCP is a small, early-stage team.
Operational simplicity matters more than cost optimization at this stage. Cloud
SQL is fully managed by GCP — no HA setup, no failover testing, no backup
pipeline, no DB patching, no operator lifecycle to manage. The team can focus
entirely on building UCP features rather than operating database infrastructure.

The 99.95% SLA, automatic failover in ~60s, continuous PITR, and self-service
restore are all provided out of the box. When deployed on GCP alongside the
rest of UCP's infrastructure, there is no cross-cloud overhead.

**Trade-off accepted:** Higher cost vs self-managed. Not a concern given the
ops savings for a small team.

---

### 2nd — CloudNativePG on K8s (Option B, K8s path)

**When:** UCP platform is deployed on CaaS (OneCloud) with a dedicated cluster
already procured, or when cost reduction is a priority over Cloud SQL.

If UCP is running on a dedicated CaaS cluster anyway, deploying PostgreSQL as
pods on the same cluster is the more logical choice. The database becomes part
of the cluster you're already operating and paying for — no separate
infrastructure, no separate billing, marginal incremental cost.

CloudNativePG significantly reduces the traditional operational burden of
self-managed PostgreSQL. A single `Cluster` CRD replaces everything —
StatefulSet, HA, failover, services, backups. It is substantially easier to
maintain compared to installing PostgreSQL manually, managing Patroni, running
etcd, and configuring HAProxy. The K8s-native tooling also means the team
already knows how to interact with it.

**Trade-off accepted:** Self-managed — the team owns backup configuration, DB
version upgrades, and recovery procedures. CloudNativePG reduces this burden
but does not eliminate it.

---

### Not recommended (Option C — DBaaS MySQL)

The confirmed absence of automatic failover is a blocker. If the primary goes
down, the DBaaS team must manually intervene — RTO is unknown and outside
UCP's control. This makes it difficult to commit to a 99.9% availability
target for the Platform DB. The engine change (MySQL instead of PostgreSQL)
also requires reopening ADR-002 with a weaker technical rationale.

Option C would only become viable if the DBaaS team confirms automatic failover
support in a future release.

---

## Cost Estimates

> **Disclaimer:** All figures are rough estimates for directional comparison
> only — not quotes. Actual costs depend on final spec, environment, and
> negotiated pricing.
>
> - **Option A**: GCP public list pricing, asia-northeast1 region. Committed
>   use discounts (1yr ~30%, 3yr ~55%) would reduce this significantly.
> - **Option B CaaS**: FY26 shared node rate ¥2,616/unit from the CaaS pricing
>   page. GKE cost is marginal on a shared cluster (node cost already paid).
> - **Option C**: FY24 DBaaS compute tier pricing. MySQL Primary-Replica may
>   bundle compute differently — needs confirmation from the DBaaS team.

**Spec basis:** The minimum reasonable production spec for UCP's load profile
(~300 concurrent users peak, simple queries, ~0.1 writes/second). This is
NOT the lowest possible spec — it includes headroom. A smaller spec (1 vCPU /
4GB) would likely work and cost roughly half, but has not been load tested.

**What's included:** Platform DB only (HA, storage). Temporal DB is excluded
— it runs as a pod inside the K8s cluster at negligible marginal cost.

**What's not included:** Network egress, backup storage, monitoring agent cost.

| | Option A — Cloud SQL | Option B — CaaS + CloudNativePG | Option C — DBaaS MySQL |
|---|---|---|---|
| **Spec** | db-custom-2-8192 (2 vCPU, 8GB), Regional HA | 2 pods × 2 vCPU / 8GB (4 units each) | S tier (4 CPU / 16GB) × 2 instances |
| **Storage** | 50GB SSD | NFS PV (CaaS) / pd-ssd (GKE) | 50GB high-spec disk |
| **Estimated monthly** | ~¥17,000 | ~¥20,928 CaaS shared / ~¥0 additional GKE shared | ~¥12,650 |
| **Billing** | GCP | OneCloud internal / GCP | OneCloud internal |

---

## Open Items Before Decision

1. **Option A — monitoring**: postgres_exporter deployed in the same GCP
   network as Cloud SQL exposes metrics to MonaaS. Needs validation in practice.

2. **Option C — RTO for manual failover**: DBaaS team involvement is required
   on primary failure. Actual response time SLA from the DBaaS team is unknown.
   Needs confirmation before Option C can be evaluated against the 99.9% target.

3. **Infrastructure decision**: Option A and Option B (GKE) costs assume UCP is
   on GCP. Option B (CaaS) and Option C assume OneCloud. The final cost and
   network dependency picture depends on OQ#3 (where to deploy UCP) in
   architecture-foundations.md.
