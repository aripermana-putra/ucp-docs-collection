---
title: "Database Engine Analysis"
space: UCP
parent_page_id: "../production-design.md"
---

# Database Engine Analysis — Platform DB

Detailed analysis supporting ADR-002. Covers the full data model, access
patterns, consistency requirements, scale characteristics, and the evaluation
of relational vs non-relational alternatives.

---

## Data Model

UCP maintains two separate database instances:

- **Platform DB** — all user-facing platform data
- **Temporal DB** — Temporal workflow state and history (Temporal owns this schema)

This document covers the Platform DB only.

### Entities

| Entity | Purpose | Retention |
|---|---|---|
| User sessions | Ties session cookie to encrypted access + refresh tokens and user identity | ~30 days or until logout |
| Tenant registry | Which OC tenants are registered and active in UCP | Indefinite |
| Quota policies | Per-tenant per-resource-type soft limits | Indefinite |
| Notification config | Slack, PagerDuty, email channel settings per tenant | Indefinite |
| Blueprint template | Service catalog — reusable templates for multi-resource provisioning | Indefinite |
| Blueprint run instance | A specific invocation of a blueprint, linked to managed resources | Lifecycle of the resource |
| Managed resource | Read model derived from etcd — resource inventory for UI and API queries | Lifecycle of the resource |
| Provisioning history | Per-resource or per-blueprint provisioning events | 3 years |
| Audit logs | Every mutating API operation across all tenants | 3 years |
| Security policies | Platform-wide rules applied at provisioning and scan time | Indefinite |
| Policy violations | Per-resource policy evaluation results from provisioning and scans | Lifecycle of the resource |

### Relationships

```
tenant_registry (tenant_rns PK)
  ├── quota_policies (tenant_rns FK)
  ├── notification_config (tenant_rns FK)
  ├── blueprint_run_instance (tenant_rns FK)
  ├── managed_resource (tenant_rns FK)
  ├── provisioning_history (tenant_rns FK)
  └── audit_logs (tenant_rns FK, denormalized)

users (id PK)
  ├── sessions (user_id FK)
  └── audit_logs (user_id FK)

blueprint_template (id PK)
  └── blueprint_run_instance (template_id FK)

managed_resource (id PK)
  ├── blueprint_run_instance (optional, if provisioned via blueprint)
  ├── provisioning_history (resource_id FK)
  └── policy_violations (resource_id FK)

security_policies (id PK)
  └── policy_violations (policy_id FK)
```

The data is relational end-to-end. Foreign keys, JOINs, and cascading deletes
are natural expressions of this structure that relational alternatives give up
for no benefit.

---

## Access Patterns

| Entity | Read pattern | Write pattern | Frequency |
|---|---|---|---|
| Sessions | Point lookup by session ID | Write on login, update on token refresh, delete on logout | Every web UI request |
| Tenant registry | Point lookup — is this tenant registered? | Write once on registration | Every provisioning request |
| Quota policies | Point lookup by tenant + resource type | Rarely — admin configuration | Every provisioning request |
| Notification config | Point lookup by tenant | Rarely — admin configuration | Every drift notification |
| Blueprint template | Point lookup by name + version | Rarely — platform team changes | Every blueprint provisioning |
| Blueprint run instance | Point lookup by ID, list by tenant | Write on create, update on status change | Per provisioning workflow |
| Managed resource | **List + filter by tenant, status, type** — primary UI query surface | Write on state change (synced from etcd) | Frequent reads, eventual write |
| Provisioning history | List by tenant or resource, time-ordered | Append only | Per provisioning event |
| Audit logs | Rare reads (compliance queries), time-range + tenant filter | **High-frequency append** — every mutation | Every mutation across all tenants |
| Security policies | Full list per provider + resource type | Rarely — platform admin | Every provisioning + scan start |
| Policy violations | List by tenant, resource, or policy | Write on provisioning violation, write on scan, clear on resolution | Low write, occasional reads |

---

## Consistency Requirements

| Entity | Requirement | Reason |
|---|---|---|
| Sessions | Strong | Stale session must not grant access |
| Tenant registry | Strong | Must not provision for unregistered tenant |
| Quota policies | Strong | Quota gate must be accurate at time of check |
| Security policies | Strong | A new policy must be visible to the next provisioning request |
| Notification config | Eventual | Delayed notification is acceptable |
| Blueprint template | Eventual | Templates change rarely |
| Blueprint run instance | Strong | Workflow state must be consistent |
| Managed resource | **Eventual** | Source of truth is etcd — DB is a derived read model |
| Provisioning history | Eventual | Append-only log, slight delay acceptable |
| Audit logs | **Zero loss** | Compliance requirement — synchronous write before API response |
| Policy violations | Eventual | Scan result from seconds ago is acceptable |

---

## Scale Analysis

| Entity | At 100 tenants (MVP) | At 1,000 tenants |
|---|---|---|
| Sessions | ~1,000 rows | ~10,000 rows |
| Tenant registry | 100 rows | 1,000 rows |
| Quota policies | ~1,000 rows | ~10,000 rows |
| Notification config | ~300 rows | ~3,000 rows |
| Blueprint templates | Tens to hundreds (shared) | Same |
| Blueprint run instances | ~22,000 rows over 3 years | ~220,000 rows |
| Managed resource | **10,000 rows** (100 resources × 100 tenants) | **100,000 rows** |
| Provisioning history | ~22,000 rows over 3 years | ~220,000 rows |
| Audit logs | **~10.9M rows over 3 years** | **~109M rows over 3 years** |
| Security policies | Tens to low hundreds | Same (platform-wide) |
| Policy violations | ~10,000 rows (one per resource per scan) | ~100,000 rows |

Audit logs are the only entity with meaningful scale concern. Every other
entity is small and handled trivially by any database at this volume.

---

## Read and Write Profile

The combined read and write profile is well within PostgreSQL's baseline
capacity without tuning.

**Peak read load:** session lookup and quota check on every API request.
Concurrent users at peak is estimated at ~300 (conservative assumption — no
measured data; for a provisioning tool where users interact periodically rather
than continuously, actual concurrency is likely lower). Both operations are
simple point lookups by primary key.

**Peak write load:** audit logs at ~10,000 events/day across all tenants —
approximately 0.1 writes/second. No entity requires the write throughput of a
wide-column store.

**Burst pattern:** the weekly security policy scan reads the full policy list
once per scan start and writes ~10,000 policy violation rows in bulk. This is
the largest single write burst in the system. Bulk insert with a single
transaction handles it cleanly.

---

## Why Relational

Three properties of the data model drive this:

**FK relationships throughout.** Every major entity references another.
Tenant-scoped queries (list all resources for tenant X, all violations for
resource Y) require JOINs or well-indexed FK lookups. An application-level
substitute in a document store would be slower, more fragile, and harder to
maintain.

**Atomicity on critical write paths.** Quota check + decrement must be atomic
— a race condition between two concurrent provisioning requests for the same
tenant must not exceed the quota. Blueprint run creation + managed resource
creation must be consistent. PostgreSQL transactions provide this natively.

**JSONB for flexible-schema fields.** Policy definitions, blueprint templates,
resource metadata, and violation details are JSON with variable structure.
JSONB columns handle this without a schema migration on every new resource type
or policy kind. The argument for a document store disappears when the relational
database already solves the flexible-schema need.

---

## Resource Desired State Lives in etcd, Not the Platform DB

A common argument for a document store in infrastructure platforms is that
resource desired state varies per resource type — a database spec looks
different from a compute spec, a Kubernetes cluster spec, or a storage bucket
spec. Schema rigidity in a relational DB would require a migration for every
new resource type.

This argument does not apply to UCP.

Crossplane XRs and MRs are the authoritative source for resource desired state,
stored as Kubernetes objects in etcd. The Platform DB holds a **read model**
(the managed resource table) for UI and API inventory queries — structured
metadata about resources, not the full spec. The variable-schema portion stays
in etcd where it belongs.

If the full desired spec ever needs to be stored in the Platform DB (for
read-model completeness or cross-cluster queries), JSONB handles it without
requiring a document store.

---

## Alternative Engines Considered

### MongoDB

MongoDB is attractive for document-like data (policy definitions, blueprint
templates) and for the variable-schema argument on resource desired state.

Both arguments are answered above: JSONB in PostgreSQL covers document-like
fields, and resource desired state lives in etcd. Adding MongoDB would mean
running a second database technology — separate backups, monitoring, connection
pooling, and on-call runbooks — for a problem PostgreSQL already solves.
MongoDB also lacks the transaction semantics required for quota enforcement and
blueprint run creation.

### Cassandra / ScyllaDB

Cassandra's argument is write throughput and horizontal scalability. The
write-heaviest entity in UCP is audit logs at ~0.1 writes/second. Cassandra is
designed for millions of writes per second. The mismatch is several orders of
magnitude.

Cassandra also has no JOINs and no multi-row transactions. Every tenant-scoped
query that currently requires a JOIN would require denormalized data or
application-level reconstruction.

### TimescaleDB

TimescaleDB is a PostgreSQL extension that adds automatic time-based
partitioning and columnar compression for time-series data. Audit logs and
policy violations have time-series characteristics that TimescaleDB handles
elegantly.

Standard PostgreSQL range partitioning by month achieves the same outcome for
the volumes in scope. TimescaleDB is worth revisiting if audit log volume at
1,000+ tenants proves difficult to manage with manual partitioning — but it
remains PostgreSQL under the hood, so the migration path is low-risk.

### Neo4j

Policy evaluation has a graph-like structure: a policy applies to resource
types, resource types belong to providers, providers are subscribed to by
tenants. Graph traversal could express this naturally.

The graph is shallow — two to three hops. PostgreSQL CTEs (Common Table
Expressions) handle recursive queries of this depth without a dedicated graph
engine. Adding Neo4j would introduce a third database technology for a query
pattern that a standard SQL CTE covers in five lines.

### Summary

| Alternative | Primary argument | Why dismissed |
|---|---|---|
| MongoDB | Flexible schema per resource type | Resource desired state lives in etcd; JSONB handles remaining flexible fields |
| Cassandra / ScyllaDB | Write throughput | ~0.1 audit writes/second makes this irrelevant; no JOINs or transactions |
| TimescaleDB | Time-series audit logs | Standard partitioning sufficient at current scale; extension available if needed |
| Neo4j | Policy evaluation graph | Two to three hop depth; PostgreSQL CTEs are sufficient |

---

## Audit Log Partitioning Strategy

Audit logs are the only entity that requires explicit scale planning.

**Partitioning:** range partition by month (`audit_logs_2026_06`,
`audit_logs_2026_07`, ...). Each month's partition is a separate physical
segment with its own indexes. Queries filtered by time range hit only the
relevant partitions, not the full table.

**Archival:** after 12 months, the partition is exported to cold object storage
(GCS) and dropped from the live database. The 3-year retention requirement is
met by the combination of live partitions (12 months) and archived exports
(remaining 24 months).

**Index strategy:** `(tenant_rns, created_at)` composite index on each
partition covers the most common query pattern (compliance queries for a
specific tenant in a time range).

The same partitioning approach applies to `policy_violations` and
`provisioning_history` given their append-only, time-ordered write patterns.

---

## Two Database Instances

Platform DB and Temporal DB are separate instances for the following reasons:

| Concern | Platform DB | Temporal DB |
|---|---|---|
| Schema ownership | UCP team | Temporal Server version drives schema migrations |
| Upgrade coupling | Independent | Temporal Server version must match schema version |
| Access patterns | Mixed reads and writes | High-frequency sequential writes (workflow history) |
| Retention policy | 3-year audit logs drive sizing | 90-day workflow retention |
| Failure scope | Platform DB down → API requests fail; Temporal continues | Temporal DB down → provisioning/drift stops; API reads still work |

A Platform DB maintenance window does not halt Temporal. A Temporal schema
migration does not risk the sessions or audit logs tables.

---

## Open Questions

1. **HA deployment model** — Cloud SQL (managed), Patroni (self-managed), or
   Crunchy PGO (K8s-native operator). Addressed separately.

2. **Multi-cluster DB sync** — if UCP runs Crossplane instances in multiple
   clouds, the managed resource read model needs a sync strategy. Options:
   central DB with cross-cluster writes, per-cluster DB with replication, or
   no read model (query each cluster's K8s API directly). Addressed when
   multi-cluster architecture is decided.

3. **Audit log archival automation** — the tooling and schedule for partition
   export to cold storage and partition drop needs to be defined before going
   to production with 3-year retention requirements.
