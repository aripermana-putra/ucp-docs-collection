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

PostgreSQL runs as pods on a dedicated CaaS cluster. **Patroni** manages the
cluster — it is a Python daemon that runs alongside each PostgreSQL process,
using the **Kubernetes API as the consensus store** (no separate etcd needed
on K8s).

```
Pod 1 (CaaS)             Pod 2 (CaaS)
┌──────────────┐         ┌──────────────┐
│  PostgreSQL  │         │  PostgreSQL  │
│  Patroni     │         │  Patroni     │
└──────┬───────┘         └──────┬───────┘
       └──────── K8s API ───────┘
                (leader election via
                 Kubernetes endpoints)
```

All Patroni daemons coordinate via the K8s API. On failover, Patroni promotes
a standby and updates the cluster state. All other nodes repoint to the new
primary.

### Stable endpoint on Kubernetes

On K8s, HAProxy is not needed. Patroni updates pod labels on failover
(`role=primary` / `role=replica`). A Kubernetes Service with a label selector
on `role=primary` automatically updates its endpoint to the new primary pod.
The application connects to the stable K8s Service DNS name — same behavior
as Cloud SQL's managed endpoint, built natively into K8s.

```
Application → postgres-primary.namespace.svc  (K8s Service, stable DNS)
                    │
                    └──► Pod with label role=primary (updated by Patroni on failover)
```

### Read replicas

**All Patroni standbys are hot standbys** — they serve reads AND act as failover
candidates simultaneously. There is no distinction between a "HA standby" and a
"read replica" at the node level.

```
Primary  ──WAL streaming──► Standby 1  ← reads + failover candidate
         ──WAL streaming──► Standby 2  ← reads + failover candidate
```

Sync vs async is configurable per standby:

- **Synchronous standby**: primary waits for confirmation before acknowledging
  writes. RPO = 0. Used for the primary failover target.
- **Asynchronous standby**: primary does not wait. Some replication lag. Used
  for read scaling or cross-DC BCP copy.

### HA — replication mechanism

PostgreSQL native **WAL streaming replication**. Every write goes to the WAL
first. The WAL is streamed continuously to standby nodes. Synchronous mode:
primary waits for at least one sync standby to confirm receipt before
acknowledging the write to the client.

### Failover mechanism

Patroni detects primary failure and promotes a standby. However, Patroni nodes
have fixed IPs. After failover, the new primary is a different IP from the old
primary. Applications would connect to the wrong node.

**Solution: HAProxy** sits in front of the cluster as a gateway layer. It polls
Patroni's REST API every 2 seconds to determine who is current primary, then
routes writes there and reads to standbys.

```
Application
     │
     ▼
  HAProxy  ←── polls Patroni REST API
  (stable endpoint)
     │
     ├── writes ──► Node 1 (current primary)
     └── reads  ──► Node 2, Node 3 (standbys)
```

On bare metal or VMs (without K8s), HAProxy is needed as a separate component
to provide a stable endpoint. On CaaS (K8s), the K8s Service handles this
natively — no HAProxy needed.

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
- **Patroni metrics** — Patroni exposes a `/metrics` endpoint.

Both feed into **MonaaS**. postgres_exporter runs in the same CaaS cluster,
same network as the database pods.

### SLA

No SLA commitment — availability is as good as you operate it. CaaS
infrastructure has its own SLA, but database-layer availability is your
responsibility.

### Key properties

| Dimension | Detail |
|---|---|
| Engine | PostgreSQL — ADR-002 stands |
| Lv2 satisfied? | Yes — Patroni, 2 pods same DC |
| Failover | Automatic via Patroni + K8s Service label selector, ~30–60s, no HAProxy needed on K8s |
| RPO | 0 with synchronous standby |
| Ops overhead | Medium — K8s operator (CloudNativePG / Zalando PGO) reduces lifecycle burden vs bare metal, but backup pipeline and DB upgrades remain your responsibility |
| PITR | Possible via pgBackRest + WAL archiving, self-managed |
| Monitoring | postgres_exporter + Patroni metrics → MonaaS |
| Network | Local to OneCloud — no cross-cloud dependency |
| Storage | NFS-backed PVs — network I/O latency, acceptable for UCP's load profile |
| Cost | ~¥20,928/month shared nodes (2 pods × 4 units × ¥2,616/unit). Marginal if on dedicated cluster already procured for UCP. |
| Internal precedent | None for PostgreSQL on CaaS |

### BMaaS reference

If CaaS is unavailable and a VMaaS exception cannot be obtained, BMaaS is the
fallback. PostgreSQL + Patroni runs on bare metal servers. Architecture and
replication are identical — only the compute unit changes.

| | CaaS (pods) | BMaaS (bare metal) |
|---|---|---|
| Min spec (Patroni HA) | 2 pods × 4 units | 2 × c7.standard |
| Monthly cost | ~¥20,928 | ~¥42,164 |
| Right-sized for UCP? | Yes | No — significant overkill |
| Stable endpoint | K8s Service (native) | HAProxy (extra component) |
| Storage | NFS PV | Local disk (better I/O) |

BMaaS delivers better raw I/O (local disk vs NFS) but at ~2× the cost and
with significant over-provisioning. Only justified if local disk performance
is a demonstrated requirement — which it is not at UCP's scale.

### References

- [Patroni documentation](https://patroni.readthedocs.io/)
- [CloudNativePG operator](https://cloudnative-pg.io/)
- [Zalando Postgres Operator](https://github.com/zalando/postgres-operator)
- [pgBackRest documentation](https://pgbackrest.org/)
- [postgres_exporter](https://github.com/prometheus-community/postgres_exporter)
- [CaaS pricing](https://onecloud.rakuten-it.com/one-docs/docs/Compute/CaaS/caas-pricing)
- [BMaaS pricing](https://onecloud.rakuten-it.com/one-docs/docs/Compute/BMaaS/bmaas-pricing)

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

| Dimension | A — Cloud SQL | B — Self-managed Patroni | C — DBaaS MySQL |
|---|---|---|---|
| **Engine** | PostgreSQL ✓ | PostgreSQL ✓ | MySQL — ADR-002 rejected |
| **Lv2 satisfied** | Yes (ZONAL) | Yes | Yes |
| **SLA** | 99.95% (Enterprise) | No commitment | 99.95% (single DC) |
| **Replication** | Synchronous (HA standby) | Synchronous (configurable) | Semi-synchronous |
| **RPO** | 0 | 0 | 0 |
| **Failover** | Automatic, ~60s, stable endpoint | Automatic via Patroni + K8s Service | **No automatic failover** — manual DBaaS intervention on primary failure |
| **Read replicas** | Yes, separate instances, async | Yes, all standbys serve reads | Yes, multiple within DC |
| **PITR** | 7 days, continuous, self-service | Manual pipeline (pgBackRest), self-managed | 30-min granularity, 7 days, **DBaaS team required** |
| **Ops overhead** | Minimal | High | Minimal |
| **Storage** | GCP SSD (managed) | NFS PV on CaaS | DBaaS managed |
| **Cost (Platform DB only)** | ~¥17,000/month | ~¥20,928/month shared; marginal on dedicated | ~¥12,650/month |
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

| | Option A — Cloud SQL | Option B — CaaS + Patroni | Option C — DBaaS MySQL |
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
