---
title: "Production Design"
space: UCP
parent_page_id: "../universal-control-plane-ucp.md"
---

# Production Design

System design specifications for the UCP production deployment.

---

## Documents

### [Architecture Foundations](./production-design/architecture-foundations.md)

Pre-design questions answered before drawing the architecture. Covers what the
system does, who uses it, data ownership boundaries, integration points,
operational ownership, NFRs, scalability dimensions, security boundaries, and
what is deferred vs decided now. Start here before reading the system design.

### [System Design](./production-design/system-design.md)

Full production architecture specification for the MVP milestone (2 cloud providers,
100 tenants, 1,000 active users). Covers NFRs, scale analysis, Kubernetes cluster
topology options with comparison by scalability, availability, blast radius, and
operational complexity, and per-component design with justification for every
technology choice including explicit reasoning for components not included.

### [Database Engine Analysis](./production-design/database-engine-analysis.md)

Detailed analysis supporting ADR-002. Full data model with entity relationships,
access patterns, consistency requirements, scale projections, audit log
partitioning strategy, and evaluation of all alternative database engines
(MongoDB, Cassandra, TimescaleDB, Neo4j) with explicit dismissal rationale.

### [Database HA Deployment Analysis](./production-design/database-ha-analysis.md)

Three-way comparison of Platform DB deployment options: Cloud SQL (GCP managed
PostgreSQL), self-managed PostgreSQL with Patroni on VMaaS, and MySQL on
OneCloud DBaaS. Covers replication mechanism, failover, read replicas, PITR,
monitoring, ops overhead, and SLA for each option. Includes open items before
a final decision can be made.

### [Architecture Diagrams](./production-design/diagrams.md)

C1 (System Context) and C2 (Container) diagrams for all three cluster topology
options: single cluster, Platform + Operations split, and three-cluster separation.

### [Scaling Architecture](./production-design/scaling-architecture.md)

C2 diagrams per scale level showing how the architecture evolves as tenant count
grows: Level 1 (single cluster, 0–500 tenants), Level 2 (Platform + Ops split,
500–2,000 tenants), and Level 3 (cluster sharding by tenant, 2,000+ tenants).
