---
title: "Database HA Deployment Analysis"
space: UCP
parent_page_id: "../production-design.md"
---

# Database HA Deployment Analysis — Platform DB

Detailed analysis of three deployment options for the Platform DB, supporting the
decision that will follow ADR-002 (database engine). If Option C is chosen, ADR-002
is rejected and the engine changes from PostgreSQL to MySQL.

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

UCP uses **MonaaS** for metrics and **EaaS** for logging. Cloud SQL metrics
live in GCP Cloud Monitoring natively. Whether these can be federated into
MonaaS needs confirmation — this is an open item specific to Option A given
that Cloud SQL sits in GCP while UCP's monitoring stack is on OneCloud.

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

## Option B — Self-managed PostgreSQL on VMaaS (OneCloud)

### Architecture

PostgreSQL runs on VMaaS VMs. **Patroni** manages the cluster — it is a Python
daemon that runs alongside each PostgreSQL process on every node, using a
consensus store (etcd or Kubernetes API) for leader election.

```
Node 1 (VM)              Node 2 (VM)              Node 3 (VM)
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  PostgreSQL  │         │  PostgreSQL  │         │  PostgreSQL  │
│  Patroni     │         │  Patroni     │         │  Patroni     │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       └─────────────────────── etcd ────────────────────┘
                         (leader election,
                          cluster state)
```

All Patroni daemons coordinate via the consensus store. On failover, Patroni
promotes a standby and updates the cluster state. All other nodes repoint to
the new primary.

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

HAProxy provides the stable endpoint that Cloud SQL builds in natively. It is
an additional component to deploy and operate.

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

### Monitoring

No managed monitoring — you own the observability stack:

- **postgres_exporter** — Prometheus exporter for PostgreSQL metrics. Exposes
  database-level metrics (connections, locks, replication lag, table stats).
- **Patroni metrics** — Patroni exposes a `/metrics` endpoint compatible with
  Prometheus.
- **node_exporter** — OS-level metrics (CPU, memory, disk) on each VM.

All three expose Prometheus-format metrics that feed into **MonaaS** for
metrics collection. Database and application logs ship to **EaaS** for log
aggregation. No distributed tracing.

### SLA

No SLA commitment — availability is as good as you operate it. The underlying
VMaaS infrastructure has its own SLA, but database-layer availability is your
responsibility.

### Key properties

| Dimension | Detail |
|---|---|
| Engine | PostgreSQL — ADR-002 stands |
| Lv2 satisfied? | Yes — Patroni, 2 nodes same DC |
| Failover | Automatic via Patroni, ~30–60s, requires HAProxy for stable endpoint |
| RPO | 0 with synchronous standby |
| Ops overhead | **High** — OS patches, DB upgrades, security patches, backup pipeline, failover testing, HAProxy operation, all manual |
| PITR | Possible via pgBackRest + WAL archiving, self-managed |
| Monitoring | postgres_exporter + Patroni metrics → MonaaS, logs → EaaS |
| Network | Local to OneCloud — no cross-cloud dependency |
| Cost | Internal billing |
| Internal precedent | None — no team running self-managed PostgreSQL on VMaaS |

### References

- [Patroni documentation](https://patroni.readthedocs.io/)
- [pgBackRest documentation](https://pgbackrest.org/)
- [postgres_exporter](https://github.com/prometheus-community/postgres_exporter)
- [HAProxy PostgreSQL routing with Patroni](https://patroni.readthedocs.io/en/latest/ha_multi_dc.html)

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

**vMotion-based automatic recovery**. VMware vMotion performs live VM migration
from the failed host to a healthy host. The migrated VM retains the same IP
address. From the application perspective:

```
Before failure:  app → 10.x.x.1 (MySQL VM on host A)
Host A fails:    vMotion migrates VM to host B, same IP retained
After failover:  app → 10.x.x.1 (MySQL VM now on host B)
```

**Connection endpoint is stable by design** — no HAProxy required, no
application reconnection to a different address. Conceptually similar to
Cloud SQL's managed endpoint.

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
this is the best-aligned option for metrics. Logs ship to EaaS. No additional
integration work required — monitoring is consistent with UCP's stack.

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
| Failover | Automatic via vMotion, stable endpoint |
| RPO | 0 (semi-synchronous replication) |
| Ops overhead | Minimal — DBaaS team manages |
| PITR | 30-min granularity, 7-day retention, **restore requires DBaaS team** |
| Monitoring | MonaaS native, logs → EaaS |
| Network | Local to OneCloud — no cross-cloud dependency |
| Cost | Internal billing (lowest) |
| Internal precedent | Strong — many Rakuten teams use OneCloud DBaaS |

### References

- [OneCloud MySQL Primary-Replica intro](https://onecloud.rakuten-it.com/one-docs/docs/Database/DBaaS/mysql-primary-replica/singledc/mysql-primary-replica-intro)
- [DBaaS MySQL Primary-Replica v3.0.0 release notes](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6597402687)
- [DBaaS manual restore operation](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6434577007)

---

## Comparison Summary

| Dimension | A — Cloud SQL | B — Self-managed Patroni | C — DBaaS MySQL |
|---|---|---|---|
| **Engine** | PostgreSQL ✓ | PostgreSQL ✓ | MySQL — ADR-002 rejected |
| **Lv2 satisfied** | Yes (ZONAL) | Yes | Yes |
| **SLA** | 99.95% (Enterprise) | No commitment | 99.95% (single DC) |
| **Replication** | Synchronous (HA standby) | Synchronous (configurable) | Semi-synchronous |
| **RPO** | 0 | 0 | 0 |
| **Failover** | Automatic, ~60s, stable endpoint | Automatic via Patroni + HAProxy | Automatic via vMotion, stable endpoint |
| **Read replicas** | Yes, separate instances, async | Yes, all standbys serve reads | Yes, multiple within DC |
| **PITR** | 7 days, continuous, self-service | Manual pipeline (pgBackRest), self-managed | 30-min granularity, 7 days, **DBaaS team required** |
| **Ops overhead** | Minimal | High | Minimal |
| **Monitoring** | GCP Cloud Monitoring (MonaaS federation TBC) | postgres_exporter → MonaaS, logs → EaaS | MonaaS native, logs → EaaS |
| **Network dependency** | Cross-interconnect if app on OneCloud | Local to OneCloud | Local to OneCloud |
| **Cost** | GCP billing | Internal billing | Internal billing (lowest) |
| **Internal precedent** | Strong (multiple teams) | None | Strong (multiple teams) |

---

## Open Items Before Decision

1. **Option A — GCP Cloud Monitoring to MonaaS**: Cloud SQL metrics live in
   GCP Cloud Monitoring natively. Needs confirmation on whether these can be
   federated into MonaaS, or whether Option A requires a separate monitoring
   path for database metrics.

2. **Option C — RTO**: What is the actual RTO for vMotion-based failover?
   vMotion is typically seconds, but exact SLA from DBaaS team is unknown.

3. **Option C — read replica endpoint**: Does each read replica have its own
   stable endpoint, or does the DBaaS team provide a load-balanced read endpoint?

4. **Infrastructure decision**: Option A's cross-interconnect dependency is only
   relevant if the Platform cluster runs on OneCloud. If the Platform cluster
   moves to GCP, Option A has no cross-cloud overhead. This decision is tied
   to OQ#3 (where to deploy UCP).
