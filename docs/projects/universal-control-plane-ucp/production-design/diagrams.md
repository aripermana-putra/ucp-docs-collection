---
title: "Architecture Diagrams"
space: UCP
parent_page_id: "../production-design.md"
---

# Architecture Diagrams

C1 (System Context) and C2 (Container) diagrams for the UCP production architecture.

The C1 diagram applies to all topology options. The C2 diagrams follow the scaling
journey — start at Level 1 and grow into the next level only when measured thresholds
are crossed. See [Architecture Foundations § Scaling Strategy](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6740191853/Architecture+Foundations#ArchitectureFoundations-Scalingstrategy)
for transition thresholds.

---

## Infrastructure Topology

High-level diagrams showing what single-cluster multi-AZ and multi-region
deployments physically look like. No UCP components shown — infrastructure
context only.

### Single Cluster — Multi-AZ

One K8s cluster with worker nodes spread across 3 availability zones within
a single region. The control plane (API server, etcd, scheduler) is managed
by the cloud provider and automatically replicated across AZs. Pods are
scheduled across worker nodes in different AZs via PodAntiAffinity.

A single zone failure loses some pods — K8s reschedules them to surviving
zones. The cluster itself stays operational.

```mermaid
graph TB
    subgraph region["Region (e.g. asia-northeast1)"]
        subgraph cp["Control Plane — managed by cloud provider (GKE/CaaS)"]
            apiserver["API Server + etcd\nReplicated across AZs automatically\nNot on worker nodes — not your responsibility"]
        end

        subgraph az_a["Availability Zone A"]
            nodes_a["Worker Nodes\nUCP pods scheduled here"]
        end

        subgraph az_b["Availability Zone B"]
            nodes_b["Worker Nodes\nUCP pods scheduled here"]
        end

        subgraph az_c["Availability Zone C"]
            nodes_c["Worker Nodes\nUCP pods scheduled here"]
        end

        apiserver --- nodes_a
        apiserver --- nodes_b
        apiserver --- nodes_c
    end
```

---

### Multi-Region — Active-Passive

Two separate K8s clusters across two regions. Each cluster is multi-AZ within
its own region. Databases replicate from Region A to Region B. Region B is a
warm standby — cluster running, databases up to date, not serving traffic.

etcd is NOT replicated cross-region. XR desired state is reconstructed on
failover — mechanism TBD [OQ#5](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6740191853/Architecture+Foundations#ArchitectureFoundations-OpenQuestions).

```mermaid
graph LR
    subgraph region_a["Region A — Primary (serves traffic)"]
        subgraph cluster_a["K8s Cluster — multi-AZ"]
            direction TB
            cp_a["Control Plane\n(managed)"]
            az_a1["AZ-a · Worker Nodes"]
            az_a2["AZ-b · Worker Nodes"]
            az_a3["AZ-c · Worker Nodes"]
            tempdb_a[("Temporal DB\nprimary\ninside cluster")]
            cp_a --- az_a1
            cp_a --- az_a2
            cp_a --- az_a3
        end
        platdb_a[("Platform DB\nprimary\ninside or outside cluster — see RFC-005")]
    end

    subgraph region_b["Region B — Standby (not serving traffic)"]
        subgraph cluster_b["K8s Cluster — multi-AZ"]
            direction TB
            cp_b["Control Plane\n(managed)"]
            az_b1["AZ-a · Worker Nodes"]
            az_b2["AZ-b · Worker Nodes"]
            az_b3["AZ-c · Worker Nodes"]
            tempdb_b[("Temporal DB\nstandby replica\ninside cluster")]
            cp_b --- az_b1
            cp_b --- az_b2
            cp_b --- az_b3
        end
        platdb_b[("Platform DB\nstandby replica\ninside or outside cluster — see RFC-005")]
    end

    secrets[("Secret Manager\nTBD — replication depends on OQ#1\ne.g. Vault needs cross-region setup, GCP Secret Manager is global")]

    platdb_a -->|"streaming replication · RPO ~1–5s"| platdb_b
    tempdb_a -->|"streaming replication · RPO ~1–5s"| tempdb_b
    region_a -. "DNS / LB failover" .-> region_b
    region_a --- secrets
    region_b --- secrets
```

---

## C1 — System Context

```mermaid
C4Context
    title System Context — Universal Control Plane (UCP)

    Person(dev, "Developer", "Provisions cloud resources via CLI or Web UI. Receives alerts and monitors resource status.")
    Person(ta, "Tenant Admin", "Approves provisioning requests and drift reconciliation. Manages team roles, quotas, and cloud credentials.")
    Person(pa, "Platform Admin", "Manages registered tenants and platform-level configuration.")

    System_Boundary(ucp_b, "Universal Control Plane") {
        System(ucp, "UCP", "Multi-cloud infrastructure provisioning, drift detection, approval workflows, RBAC, and quota management.")
    }

    System_Ext(gcp, "Google Cloud Platform", "Provisions and manages Cloud SQL, GKE clusters, GCE instances, and GCS buckets.")
    System_Ext(omnia, "OneCloud / Omnia", "Provisions and manages Omnia DBaaS resources.")
    System_Ext(horizon, "Horizon / Keycloak", "OIDC authentication. Tenant membership and role data via ROC realm JWT groups.")
    System_Ext(apic, "API-C", "API Gateway. Inbound routing, rate limiting, and correlation ID injection.")
    System_Ext(pd, "PagerDuty", "Receives incident alerts from UCP.")
    System_Ext(slack, "Slack", "Receives notifications from UCP.")
    System_Ext(email, "Email", "Receives notifications from UCP.")

    Rel(dev, apic, "Provisions resources, views status, approves drift via CLI or Web UI", "HTTPS")
    Rel(ta, apic, "Approves workflows, manages roles and credentials", "HTTPS")
    Rel(pa, apic, "Manages platform configuration", "HTTPS")
    Rel(apic, ucp, "Routes inbound requests", "HTTPS")

    Rel(ucp, gcp, "Provisions and reconciles GCP resources", "GCP REST APIs")
    Rel(ucp, omnia, "Provisions and reconciles Omnia DBaaS resources", "Omnia REST API")
    Rel(ucp, horizon, "Authenticates users, syncs tenant memberships and roles", "OIDC / Horizon REST API")
    Rel(ucp, pd, "Sends alerts", "PagerDuty Events API v2")
    Rel(ucp, slack, "Sends notifications", "Incoming Webhook")
    Rel(ucp, email, "Sends notifications", "SMTP")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

---

## C2 — Level 1: Single Cluster (0–500 tenants)

All UCP workloads run on a single multi-AZ Kubernetes cluster.
Start here. Transition to Level 2 only when measured resource contention justifies it.

> **Cluster spec:** 3 × (4 vCPU, 16GB RAM) across 3 AZs. Cluster Autoscaler
> adds nodes under load. See [Architecture Foundations § Initial Cluster Spec](./architecture-foundations.md#initial-cluster-spec-estimated).

```mermaid
graph TB
    user(["User
Developer · Tenant Admin · Platform Admin"])

    apic["API-C
API Gateway"]
    keycloak["Keycloak
OIDC"]
    coredata["Core Data API"]
    notif["PagerDuty · Slack · Email"]
    gcp["GCP
Cloud SQL · GKE · GCE · GCS"]
    roc["ROC / OneCloud
Omnia DBaaS"]
    secrets["Secret Manager
TBD"]

    subgraph cluster["K8s Cluster — multi-AZ"]
        ingress["Ingress"]
        api["API Server
Go / Echo"]
        platdb[("Platform DB
PostgreSQL")]
        temporal["Temporal Server
Temporal OSS"]
        tempdb[("Temporal DB
PostgreSQL")]
        prov_w["Provisioning Worker
Go / Temporal SDK"]
        drift_w["Drift Worker
Go / Temporal SDK · KEDA"]
        crossplane["Crossplane + Providers
provider-gcp-* · provider-roc"]
        eso["ESO"]
        keda["KEDA"]
    end

    user --> apic --> ingress --> api

    api --> platdb & temporal & crossplane
    api --> keycloak & coredata & secrets

    temporal --> tempdb
    prov_w --> temporal & crossplane
    drift_w --> temporal & crossplane & platdb & notif
    keda --> temporal
    eso --> secrets

    crossplane --> gcp & roc
```

---

## C2 — Level 2: Platform + Ops Split (500–2,000 tenants)

Platform-facing components (API, auth, data) are separated from operations components
(Crossplane, Temporal, workers). Two clusters per environment.

Triggered when: Crossplane MR write churn shows measurable correlation with API
server read latency, or provider pod memory pressure is consistently high.

> **Cross-cluster connections:** API Server → Temporal (gRPC via Internal LB),
> API Server → Ops K8s API (kubeconfig, scoped ServiceAccount),
> Drift Worker → Platform DB (SQL), ESO → Secret Manager (HTTPS).

```mermaid
graph TB
    user(["User
Developer · Tenant Admin · Platform Admin"])

    apic["API-C
API Gateway"]
    keycloak["Keycloak
OIDC"]
    coredata["Core Data API"]
    notif["PagerDuty · Slack · Email"]
    gcp["GCP
Cloud SQL · GKE · GCE · GCS"]
    roc["ROC / OneCloud
Omnia DBaaS"]
    secrets["Secret Manager
TBD"]

    subgraph plat["Platform Cluster — multi-AZ"]
        ingress["Ingress"]
        api["API Server
Go / Echo"]
        platdb[("Platform DB
PostgreSQL")]
    end

    subgraph ops["Ops Cluster — multi-AZ"]
        temporal["Temporal Server
Temporal OSS"]
        tempdb[("Temporal DB
PostgreSQL")]
        prov_w["Provisioning Worker
Go / Temporal SDK"]
        drift_w["Drift Worker
Go / Temporal SDK · KEDA"]
        crossplane["Crossplane + Providers
provider-gcp-* · provider-roc"]
        eso["ESO"]
        keda["KEDA"]
    end

    user --> apic --> ingress --> api

    api --> platdb & secrets & keycloak & coredata
    api -->|"cross-cluster"| temporal & crossplane

    temporal --> tempdb
    prov_w --> temporal & crossplane
    drift_w --> temporal & crossplane & notif
    drift_w -->|"cross-cluster"| platdb
    keda --> temporal
    eso -->|"cross-cluster"| secrets

    crossplane --> gcp & roc
```

---

## C2 — Level 3: Cluster Sharding (2,000+ tenants)

Ops cluster is sharded by tenant. Each shard runs an independent set of Temporal
task queues and Crossplane providers. The API server contains a shard router that
maps tenant → shard via consistent hashing.

Triggered when: single Ops cluster Crossplane provider memory or Temporal throughput
becomes a measurable bottleneck at 2,000+ tenants.

> **Shard router:** lives inside the API server. Maps tenant ID → shard index via
> consistent hashing. Each shard has its own Temporal task queue namespace and
> Crossplane provider pods. Adding a shard requires re-balancing the tenant→shard
> mapping and migrating in-flight workflows.

```mermaid
graph TB
    user(["User
Developer · Tenant Admin · Platform Admin"])

    apic["API-C
API Gateway"]
    keycloak["Keycloak
OIDC"]
    coredata["Core Data API"]
    notif["PagerDuty · Slack · Email"]
    gcp["GCP
Cloud SQL · GKE · GCE · GCS"]
    roc["ROC / OneCloud
Omnia DBaaS"]
    secrets["Secret Manager
TBD"]

    subgraph plat["Platform Cluster — multi-AZ"]
        ingress["Ingress"]
        api["API Server + Shard Router
Go / Echo"]
        platdb[("Platform DB
PostgreSQL")]
    end

    subgraph shard_a["Ops Cluster — Shard A (tenants 1–N)"]
        temporal_a["Temporal Server"]
        tempdb_a[("Temporal DB")]
        workers_a["Provisioning + Drift Workers
Go / Temporal SDK · KEDA"]
        crossplane_a["Crossplane + Providers"]
        eso_keda_a["ESO + KEDA"]
    end

    subgraph shard_b["Ops Cluster — Shard B (tenants N+1–M)"]
        temporal_b["Temporal Server"]
        tempdb_b[("Temporal DB")]
        workers_b["Provisioning + Drift Workers
Go / Temporal SDK · KEDA"]
        crossplane_b["Crossplane + Providers"]
        eso_keda_b["ESO + KEDA"]
    end

    user --> apic --> ingress --> api
    api --> platdb & secrets & keycloak & coredata
    api -->|"Shard A tenants"| temporal_a & crossplane_a
    api -->|"Shard B tenants"| temporal_b & crossplane_b

    temporal_a --> tempdb_a
    workers_a --> temporal_a & crossplane_a & notif
    workers_a -->|"cross-cluster"| platdb
    eso_keda_a -->|"cross-cluster"| secrets
    crossplane_a --> gcp & roc

    temporal_b --> tempdb_b
    workers_b --> temporal_b & crossplane_b & notif
    workers_b -->|"cross-cluster"| platdb
    eso_keda_b -->|"cross-cluster"| secrets
    crossplane_b --> gcp & roc
```

---

## Multi-Region Active-Passive (BCP Lv4 - If needed)

Two clusters across two regions. Region A is the primary and serves all traffic.
Region B is a warm standby — all components deployed and running, databases
replicated, but not serving requests. On Region A failure, traffic is redirected
to Region B via DNS/load balancer failover and standby databases are promoted.

> **What replicates:**
> - Platform DB — PostgreSQL streaming replication, Region A → Region B. RPO ~1–5s.
> - Temporal DB — PostgreSQL streaming replication, Region A → Region B. RPO ~1–5s.
> - Secret Manager — global service, no replication needed.
>
> **What does NOT replicate:**
> - etcd (K8s state) — XR/MR objects are NOT replicated cross-region. On failover,
>   XR desired state is re-applied to Region B cluster and Crossplane reconstructs
>   MR state via Observe() against actual cloud resources. Mechanism TBD — see
>   [OQ#5](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6740191853/Architecture+Foundations#ArchitectureFoundations-OpenQuestions)).
>
> **In-flight Temporal workflows** at time of failure are lost and must be
> re-submitted after failover. Acceptable at UCP's workflow volume (~4–5 active at any moment).
>
> **Failover automation level TBD** — fully automated, semi-automated (human triggers,
> automated execution), or manual runbook. See [OQ#5](https://confluence.rakuten-it.com/confluence/spaces/UCP/pages/6740191853/Architecture+Foundations#ArchitectureFoundations-OpenQuestions).

```mermaid
graph LR
    user(["User"])
    apic["API-C + Global LB
Routes to Region A · Failover to Region B"]
    keycloak["Keycloak · Core Data"]
    notif["PagerDuty · Slack · Email"]
    gcp["GCP"]
    roc["ROC / OneCloud"]
    secrets["Secret Manager
TBD — see OQ#1 for replication model"]

    subgraph region_a["Region A — Primary (serves traffic)"]
        ingress_a["Ingress"]
        api_a["API Server"]
        platdb_a[("Platform DB
primary")]
        temporal_a["Temporal Server"]
        tempdb_a[("Temporal DB
primary · inside cluster")]
        workers_a["Provisioning + Drift Workers"]
        crossplane_a["Crossplane + Providers
idle on failover — reconstructs via Observe()"]
        eso_keda_a["ESO + KEDA"]
    end

    subgraph region_b["Region B — Standby (not serving traffic)"]
        ingress_b["Ingress"]
        api_b["API Server
idle until failover"]
        platdb_b[("Platform DB
standby replica")]
        temporal_b["Temporal Server
idle until failover"]
        tempdb_b[("Temporal DB
standby replica · inside cluster")]
        workers_b["Provisioning + Drift Workers
idle until failover"]
        crossplane_b["Crossplane + Providers
idle until failover"]
        eso_keda_b["ESO + KEDA"]
    end

    user --> apic
    apic -->|"normal"| ingress_a
    apic -. "failover only" .-> ingress_b

    ingress_a --> api_a
    api_a --> platdb_a & temporal_a & crossplane_a & keycloak & secrets
    temporal_a --> tempdb_a
    workers_a --> temporal_a & crossplane_a & notif & platdb_a
    eso_keda_a --> secrets
    crossplane_a --> gcp & roc

    platdb_a -->|"streaming replication · RPO ~1–5s"| platdb_b
    tempdb_a -->|"streaming replication · RPO ~1–5s"| tempdb_b

    ingress_b --> api_b
    api_b --> platdb_b & temporal_b & crossplane_b & keycloak & secrets
    temporal_b --> tempdb_b
    workers_b --> temporal_b & crossplane_b & notif & platdb_b
    eso_keda_b --> secrets
    crossplane_b --> gcp & roc
```

---

## Alternative — Three-Cluster Separation (not recommended at UCP scale)

Platform, Crossplane, and Temporal each run on dedicated clusters. Maximum blast
radius isolation and independent scaling per tier. Not recommended for UCP's current
and foreseeable scale — the operational overhead of three clusters outweighs the
benefit. Documented here for reference only.

> Revisit only if Temporal and Crossplane show independent resource contention that
> cannot be resolved by vertical scaling within the Ops cluster.

```mermaid
graph TB
    user(["User
Developer · Tenant Admin · Platform Admin"])

    apic["API-C
API Gateway"]
    keycloak["Keycloak
OIDC"]
    coredata["Core Data API"]
    notif["PagerDuty · Slack · Email"]
    gcp["GCP
Cloud SQL · GKE · GCE · GCS"]
    roc["ROC / OneCloud
Omnia DBaaS"]
    secrets["Secret Manager
TBD"]

    subgraph plat["Platform Cluster — multi-AZ"]
        ingress["Ingress"]
        api["API Server
Go / Echo"]
        platdb[("Platform DB
PostgreSQL")]
    end

    subgraph cp["Crossplane Cluster — multi-AZ"]
        crossplane["Crossplane + Providers
provider-gcp-* · provider-roc"]
        prov_w["Provisioning Worker
Go / Temporal SDK"]
        drift_w["Drift Worker
Go / Temporal SDK · KEDA"]
        eso["ESO"]
        keda["KEDA"]
    end

    subgraph tc["Temporal Cluster — multi-AZ"]
        temporal["Temporal Server
Temporal OSS"]
        tempdb[("Temporal DB
PostgreSQL")]
    end

    user --> apic --> ingress --> api
    api --> platdb & secrets & keycloak & coredata
    api -->|"cross-cluster"| temporal & crossplane

    temporal --> tempdb
    prov_w -->|"cross-cluster"| temporal
    prov_w --> crossplane
    drift_w -->|"cross-cluster"| temporal & platdb
    drift_w --> crossplane & notif
    keda -->|"cross-cluster"| temporal
    eso -->|"cross-cluster"| secrets

    crossplane --> gcp & roc
```
