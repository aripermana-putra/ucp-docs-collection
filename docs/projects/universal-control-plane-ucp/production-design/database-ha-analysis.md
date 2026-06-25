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

## Option B — Self-managed PostgreSQL on CaaS (OneCloud)

VMaaS production is currently restricted to TAM and CPSD service providers
(DBaaS, CaaS, LBaaS, MonaaS, etc.) and is not available to other users.
An exception may be possible for UCP — worth exploring with the VMaaS team.
Until then, the available compute options are CaaS and BMaaS. CaaS is the
primary path. BMaaS is documented at the end of this section for reference.

### Architecture

PostgreSQL runs as pods managed by **CloudNativePG** — a Kubernetes-native
operator that implements HA, failover, replication, and backup management
directly. No Patroni. No separate HA tool. CloudNativePG is a CNCF sandbox
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

CloudNativePG creates:
- A StatefulSet with correct pod configuration
- `ucp-platform-db-rw` Service → primary only (writes)
- `ucp-platform-db-ro` Service → replicas only (reads)
- PVCs for each pod
- RBAC for K8s API access
- Manages label updates on failover automatically

### HA — replication mechanism

PostgreSQL native **WAL streaming replication**, configured and managed by
CloudNativePG. Synchronous replication (RPO = 0) is the default for the
standby — primary waits for standby acknowledgment before confirming writes.

### Failover mechanism

CloudNativePG detects primary failure via K8s liveness probes and promotes
the standby automatically. The `-rw` Service endpoint updates within seconds
— no HAProxy, no separate routing component.

```
Application → ucp-platform-db-rw (K8s Service, stable DNS)
                    │
                    └──► primary pod (CloudNativePG updates on failover)
```

### Read replicas

The `-ro` Service routes to standby pods. All standbys serve reads and act
as failover candidates simultaneously.

### Backup and PITR

No managed backup — you own the entire backup pipeline:

- **pgBackRest** (recommended): full, incremental, and differential backups
  with PITR via WAL archiving. Industry standard for PostgreSQL backup.
- **pg_dump**: logical backups, no PITR.
- Backup storage: STaaS or GCS, configured by you.
- Retention: configured by you.
- Restore: self-service, but entirely manual — you run the restore procedure.

PITR is possible with WAL archiving (pgBackRest), but you configure the
archive destination, test restores, and monitor backup health yourself.

### Storage consideration on CaaS

CaaS supports NFS-backed and STaaS persistent volumes. Network-attached storage
adds I/O latency compared to local SSD. For UCP's load profile (~0.1
writes/second, simple queries), NFS latency is unlikely to be noticeable in
practice. See the trade-off section at the end of this document for the broader
debate on databases on Kubernetes.

### Monitoring

- **postgres_exporter** — connects to PostgreSQL, queries internal statistics
  views, exposes Prometheus-format metrics.
- **CloudNativePG** — exposes its own `/metrics` endpoint covering cluster
  health, replication lag, and failover events.

Both feed into **MonaaS**. Both run in the same cluster as the database pods.

### SLA

No SLA commitment — availability is as good as you operate it. CaaS
infrastructure has its own SLA, but database-layer availability is your
responsibility.

### Key properties

| Dimension | Detail |
|---|---|
| Engine | PostgreSQL — ADR-002 stands |
| Lv2 satisfied? | Yes — CloudNativePG, 2 pods same DC |
| Failover | Automatic via CloudNativePG, ~30s, stable endpoint via K8s Service |
| RPO | 0 with synchronous standby |
| Ops overhead | Medium — CloudNativePG handles HA, failover, and replication. Backup pipeline configuration and DB version upgrades remain your responsibility. |
| PITR | CloudNativePG has built-in backup and PITR via `ScheduledBackup` CRD. Storage target (STaaS/GCS) configured by you. |
| Monitoring | postgres_exporter + CloudNativePG metrics → MonaaS |
| Network | Local to OneCloud — no cross-cloud dependency |
| Storage | NFS-backed PVs — network I/O latency, acceptable for UCP's load profile |
| Cost | ~¥20,928/month shared nodes (2 pods × 4 units × ¥2,616/unit). Marginal if on dedicated cluster already procured for UCP. |
| Internal precedent | None for PostgreSQL on CaaS |

### GKE sub-option

If UCP's platform cluster is deployed on GCP (GKE), PostgreSQL managed by
CloudNativePG can run on the same GKE cluster instead of Cloud SQL. The
architecture is identical to CaaS — same operator, same K8s Service for
stable endpoint. The key difference is storage.

**Storage advantage over CaaS:** GKE supports Persistent Disk SSD (pd-ssd),
which is block storage with low, consistent I/O latency. No NFS overhead.
Comparable to bare metal for database workloads.

**Cost:** If PostgreSQL pods run on existing GKE nodes already provisioned for
UCP's other workloads, the cost is marginal — you're consuming spare capacity
on nodes you're already paying for. If dedicated nodes are needed:

- 2 × `n2-standard-2` (2 vCPU, 8GB): ~$48/month each → ~$96/month ≈ **¥14,400/month**
- 50GB pd-ssd storage × 2: ~$17/month ≈ ¥2,550/month
- **Total: ~¥16,950/month** — slightly cheaper than Cloud SQL (~¥17,000) with better I/O, but self-managed

**Trade-off vs Cloud SQL:** Cloud SQL at roughly the same cost gives you fully
managed HA, automated backups, PITR, and zero ops overhead. PostgreSQL on GKE
costs roughly the same but you own operations. The cost saving is minimal —
the decision comes down to whether self-managed is acceptable, not cost.

| | CaaS + CloudNativePG | GKE + CloudNativePG | Cloud SQL |
|---|---|---|---|
| Storage | NFS (network I/O) | pd-ssd (block I/O) | GCP SSD (managed) |
| Cost | ~¥20,928 | ~¥16,950 (dedicated) / marginal | ~¥17,000 |
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
| Monthly cost | ~¥20,928 | ~¥16,950 | ~¥63,246 (3 nodes, HA) |
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
| **Ops overhead** | Minimal | High | Minimal |
| **Storage** | GCP SSD (managed) | NFS PV (CaaS) / pd-ssd (GKE) | DBaaS managed |
| **Cost (Platform DB only)** | ~¥17,000/month | ~¥20,928 CaaS shared / ~¥16,950 GKE dedicated / marginal on existing cluster | ~¥12,650/month |
| **Monitoring** | GCP Cloud Monitoring (MonaaS federation TBC) | postgres_exporter → MonaaS | MonaaS native |
| **Network dependency** | Cross-interconnect if app on OneCloud | Local to OneCloud | Local to OneCloud |
| **Cost** | GCP billing | Internal billing | Internal billing (lowest) |
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

### Verdict for UCP

UCP's database workload is not I/O intensive — ~0.1 writes/second, simple
point lookups, ~300 concurrent users peak. The performance argument is
irrelevant at this scale. NFS-backed PVs on CaaS will not be a bottleneck.

Running PostgreSQL on CaaS with CloudNativePG or Zalando PGO is a reasonable
production choice for UCP. The operator reduces lifecycle overhead, the K8s
Service provides a stable endpoint without HAProxy, and the workload fits
comfortably within standard CaaS node capacity.

**References:**
- [Google Cloud: To run or not to run a database on Kubernetes](https://cloud.google.com/blog/products/databases/to-run-or-not-to-run-a-database-on-kubernetes-what-to-consider)
- [CockroachDB: Running databases on Kubernetes](https://www.cockroachlabs.com/blog/kubernetes-databases/)

---

## Cost Estimates

> All figures are estimates. Option A based on GCP public list pricing
> (committed use discounts would reduce it ~30–55%). Option B CaaS based on
> FY26 shared node rate ¥2,616/unit. Option C based on FY24 DBaaS compute
> tier pricing. MySQL Primary-Replica pricing needs confirmation from the
> DBaaS team as it may bundle compute differently.

**Spec used:** 2 vCPU / 8GB RAM, 50GB storage — appropriate for UCP's load
profile (300 concurrent users, simple queries, ~0.1 writes/second).

**Temporal DB is not included** — it runs as a pod inside the K8s cluster
(same cluster as Temporal Server) at negligible marginal cost.

| | Option A — Cloud SQL | Option B — CaaS + CloudNativePG | Option C — DBaaS MySQL |
|---|---|---|---|
| **Compute** | db-custom-2-8192, Regional HA | 2 pods × 4 units × ¥2,616 | S tier (4 CPU/16GB) × 2 instances |
| **Storage** | 50GB SSD | NFS PV | 50GB high-spec disk |
| **Estimated monthly** | ~¥17,000 | ~¥20,928 (shared nodes) | ~¥12,650 |
| **On dedicated cluster** | N/A | Marginal | N/A |
| **Billing** | GCP | OneCloud internal | OneCloud internal |

---

## Open Items Before Decision

1. **Option A — GCP Cloud Monitoring to MonaaS**: Cloud SQL metrics live in
   GCP Cloud Monitoring natively. Needs confirmation on whether these can be
   federated into MonaaS, or whether Option A requires a separate monitoring
   path for database metrics.

3. **Infrastructure decision**: Option A's cross-interconnect dependency is only
   relevant if the Platform cluster runs on OneCloud. If the Platform cluster
   moves to GCP, Option A has no cross-cloud overhead. This decision is tied
   to OQ#3 in architecture foundation doc (where to deploy UCP).
