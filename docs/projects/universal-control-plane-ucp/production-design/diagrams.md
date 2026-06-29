---
title: "Architecture Diagrams"
space: UCP
parent_page_id: "../production-design.md"
---

# Architecture Diagrams

C1 (System Context) and C2 (Container) diagrams for the UCP production architecture.

The C1 diagram applies to all topology options. The C2 diagrams follow the scaling
journey — start at Level 1 and grow into the next level only when measured thresholds
are crossed. See [Architecture Foundations § Scaling Strategy]()
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
failover — mechanism TBD (OQ#5).

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
C4Container
    title C2 — Level 1: Single Cluster (0–500 tenants)

    Person(user, "User", "Developer / Tenant Admin / Platform Admin")

    System_Ext(apic, "API-C", "API Gateway. Inbound routing, rate limiting, correlation ID.")
    System_Ext(gcp, "GCP", "Cloud SQL, GKE, GCE, GCS")
    System_Ext(roc, "ROC / OneCloud", "Omnia DBaaS and other ROC services")
    System_Ext(keycloak, "Keycloak", "OIDC authentication")
    System_Ext(coredata, "Core Data API", "Tenant membership and role data")
    System_Ext(notif, "PagerDuty / Slack / Email", "Notification channels for UCP events")

    Container_Boundary(cluster, "K8s Cluster — multi-AZ") {

        Container(ingress, "Ingress", "K8s Ingress", "TLS termination, rate limiting, load balancing")
        Container(api, "API Server", "Go / Echo", "REST API. Auth, RBAC, quota, workflow submission.")

        ContainerDb(platdb, "Platform DB", "PostgreSQL", "Sessions, RBAC, audit logs, quota, notification config")
        Container(secrets, "Secret Manager", "TBD", "Cloud provider credentials and platform secrets")

        Container(temporal, "Temporal Server", "Temporal OSS", "Workflow orchestration. Frontend, History, Matching — all HA.")
        ContainerDb(tempdb, "Temporal DB", "PostgreSQL", "Workflow state, history, visibility store")

        Container(prov_w, "Provisioning Worker", "Go / Temporal SDK", "Executes provisioning workflows.")
        Container(drift_w, "Drift Worker", "Go / Temporal SDK, KEDA", "Executes drift scan and approval workflows. Scales 1–10 replicas.")

        Container(crossplane, "Crossplane + Providers", "Crossplane, provider-gcp-*, provider-roc", "Reconciles XRs and MRs against cloud provider state.")
        Container(eso, "ESO", "External Secrets Operator", "Syncs credentials from Secret Manager to K8s Secrets for Crossplane.")
        Container(keda, "KEDA", "KEDA", "Scales drift workers on Temporal queue depth.")
    }

    Rel(user, apic, "HTTPS")
    Rel(apic, ingress, "HTTPS")
    Rel(ingress, api, "HTTP")

    Rel(api, platdb, "SQL — sessions, RBAC, audit, quota")
    Rel(api, secrets, "read credentials")
    Rel(api, temporal, "gRPC — submit and query workflows")
    Rel(api, crossplane, "K8s API — create, list, delete XRs")
    Rel(api, keycloak, "OIDC — auth, token validation")
    Rel(api, coredata, "REST — tenant sync at login")

    Rel(temporal, tempdb, "SQL — workflow state")
    Rel(prov_w, temporal, "gRPC — poll provisioning queue")
    Rel(prov_w, crossplane, "K8s API — apply XR, poll XR status")
    Rel(drift_w, temporal, "gRPC — poll drift queue")
    Rel(drift_w, crossplane, "K8s API — LIST MRs, PATCH managementPolicies")
    Rel(drift_w, platdb, "SQL — read notification config")
    Rel(drift_w, notif, "HTTP")
    Rel(keda, temporal, "gRPC — read queue depth")
    Rel(eso, secrets, "read credentials")

    Rel(crossplane, gcp, "REST APIs — provision and observe")
    Rel(crossplane, roc, "REST API — provision and observe")

    UpdateLayoutConfig($c4ShapeInRow="4", $c4BoundaryInRow="1")
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
C4Container
    title C2 — Level 2: Platform + Ops Split (500–2,000 tenants)

    Person(user, "User", "Developer / Tenant Admin / Platform Admin")

    System_Ext(apic, "API-C", "API Gateway. Inbound routing, rate limiting, correlation ID.")
    System_Ext(gcp, "GCP", "Cloud SQL, GKE, GCE, GCS")
    System_Ext(roc, "ROC / OneCloud", "Omnia DBaaS and other ROC services")
    System_Ext(keycloak, "Keycloak", "OIDC authentication")
    System_Ext(coredata, "Core Data API", "Tenant membership and role data")
    System_Ext(notif, "PagerDuty / Slack / Email", "Notification channels for UCP events")

    Container_Boundary(plat, "Platform Cluster — multi-AZ") {

        Container(ingress, "Ingress", "K8s Ingress", "TLS termination, rate limiting, load balancing")
        Container(api, "API Server", "Go / Echo", "REST API. Auth, RBAC, quota, workflow submission.")
        ContainerDb(platdb, "Platform DB", "PostgreSQL", "Sessions, RBAC, audit logs, quota, notification config")
        Container(secrets, "Secret Manager", "TBD", "Cloud provider credentials and platform secrets")
    }

    Container_Boundary(ops, "Ops Cluster — multi-AZ") {

        Container(temporal, "Temporal Server", "Temporal OSS", "Workflow orchestration. Frontend, History, Matching — all HA.")
        ContainerDb(tempdb, "Temporal DB", "PostgreSQL", "Workflow state, history, visibility store")

        Container(prov_w, "Provisioning Worker", "Go / Temporal SDK", "Executes provisioning workflows.")
        Container(drift_w, "Drift Worker", "Go / Temporal SDK, KEDA", "Executes drift scan and approval workflows.")

        Container(crossplane, "Crossplane + Providers", "Crossplane, provider-gcp-*, provider-roc", "Reconciles XRs and MRs against cloud provider state.")
        Container(eso, "ESO", "External Secrets Operator", "Syncs credentials from Secret Manager to K8s Secrets for Crossplane.")
        Container(keda, "KEDA", "KEDA", "Scales drift workers on Temporal queue depth.")
    }

    Rel(user, apic, "HTTPS")
    Rel(apic, ingress, "HTTPS")
    Rel(ingress, api, "HTTP")

    Rel(api, platdb, "SQL")
    Rel(api, secrets, "read credentials")
    Rel(api, temporal, "gRPC — cross-cluster via Internal LB")
    Rel(api, crossplane, "K8s API — cross-cluster kubeconfig")
    Rel(api, keycloak, "OIDC")
    Rel(api, coredata, "REST")

    Rel(temporal, tempdb, "SQL — in-cluster")
    Rel(prov_w, temporal, "gRPC — in-cluster")
    Rel(prov_w, crossplane, "K8s API — in-cluster")
    Rel(drift_w, temporal, "gRPC — in-cluster")
    Rel(drift_w, crossplane, "K8s API — in-cluster")
    Rel(drift_w, platdb, "SQL — cross-cluster (notification config)")
    Rel(drift_w, notif, "HTTP")
    Rel(keda, temporal, "gRPC — in-cluster")
    Rel(eso, secrets, "HTTPS — cross-cluster")

    Rel(crossplane, gcp, "REST APIs")
    Rel(crossplane, roc, "REST API")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
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
C4Container
    title C2 — Level 3: Cluster Sharding (2,000+ tenants)

    Person(user, "User", "Developer / Tenant Admin / Platform Admin")

    System_Ext(apic, "API-C", "API Gateway. Inbound routing, rate limiting, correlation ID.")
    System_Ext(gcp, "GCP", "Cloud SQL, GKE, GCE, GCS")
    System_Ext(roc, "ROC / OneCloud", "Omnia DBaaS and other ROC services")
    System_Ext(keycloak, "Keycloak", "OIDC authentication")
    System_Ext(coredata, "Core Data API", "Tenant membership and role data")
    System_Ext(notif, "PagerDuty / Slack / Email", "Notification channels for UCP events")

    Container_Boundary(plat, "Platform Cluster — multi-AZ") {

        Container(ingress, "Ingress", "K8s Ingress", "TLS termination, rate limiting, load balancing")
        Container(api, "API Server + Shard Router", "Go / Echo", "REST API. Routes tenant requests to the correct Ops shard via consistent hashing.")
        ContainerDb(platdb, "Platform DB", "PostgreSQL", "Sessions, RBAC, audit logs, quota, notification config")
        Container(secrets, "Secret Manager", "TBD", "Cloud provider credentials and platform secrets")
    }

    Container_Boundary(shard_a, "Ops Cluster — Shard A (tenants 1–N)") {

        Container(temporal_a, "Temporal Server", "Temporal OSS", "Shard A task queues.")
        ContainerDb(tempdb_a, "Temporal DB", "PostgreSQL", "Shard A workflow state")
        Container(workers_a, "Provisioning + Drift Workers", "Go / Temporal SDK, KEDA", "Poll Shard A task queues.")
        Container(crossplane_a, "Crossplane + Providers", "Crossplane, provider-gcp-*, provider-roc", "Manages Shard A tenant resources.")
        Container(eso_keda_a, "ESO + KEDA", "ESO, KEDA", "Secret sync and worker autoscaling for Shard A.")
    }

    Container_Boundary(shard_b, "Ops Cluster — Shard B (tenants N+1–M)") {

        Container(temporal_b, "Temporal Server", "Temporal OSS", "Shard B task queues.")
        ContainerDb(tempdb_b, "Temporal DB", "PostgreSQL", "Shard B workflow state")
        Container(workers_b, "Provisioning + Drift Workers", "Go / Temporal SDK, KEDA", "Poll Shard B task queues.")
        Container(crossplane_b, "Crossplane + Providers", "Crossplane, provider-gcp-*, provider-roc", "Manages Shard B tenant resources.")
        Container(eso_keda_b, "ESO + KEDA", "ESO, KEDA", "Secret sync and worker autoscaling for Shard B.")
    }

    Rel(user, apic, "HTTPS")
    Rel(apic, ingress, "HTTPS")
    Rel(ingress, api, "HTTP")

    Rel(api, platdb, "SQL")
    Rel(api, secrets, "read credentials")
    Rel(api, keycloak, "OIDC")
    Rel(api, coredata, "REST")

    Rel(api, temporal_a, "gRPC — Shard A tenants")
    Rel(api, crossplane_a, "K8s API — Shard A tenants")
    Rel(api, temporal_b, "gRPC — Shard B tenants")
    Rel(api, crossplane_b, "K8s API — Shard B tenants")

    Rel(temporal_a, tempdb_a, "SQL")
    Rel(workers_a, temporal_a, "gRPC — in-cluster")
    Rel(workers_a, crossplane_a, "K8s API — in-cluster")
    Rel(workers_a, platdb, "SQL — cross-cluster (notification config)")
    Rel(workers_a, notif, "HTTP")
    Rel(eso_keda_a, secrets, "HTTPS — cross-cluster")
    Rel(crossplane_a, gcp, "REST APIs")
    Rel(crossplane_a, roc, "REST API")

    Rel(temporal_b, tempdb_b, "SQL")
    Rel(workers_b, temporal_b, "gRPC — in-cluster")
    Rel(workers_b, crossplane_b, "K8s API — in-cluster")
    Rel(workers_b, platdb, "SQL — cross-cluster (notification config)")
    Rel(workers_b, notif, "HTTP")
    Rel(eso_keda_b, secrets, "HTTPS — cross-cluster")
    Rel(crossplane_b, gcp, "REST APIs")
    Rel(crossplane_b, roc, "REST API")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
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
>   [OQ#5]).
>
> **In-flight Temporal workflows** at time of failure are lost and must be
> re-submitted after failover. Acceptable at UCP's workflow volume (~4–5 active at any moment).
>
> **Failover automation level TBD** — fully automated, semi-automated (human triggers,
> automated execution), or manual runbook. See [OQ#5]().

```mermaid
C4Container
    title Multi-Region Active-Passive (BCP Lv4)

    Person(user, "User", "Developer / Tenant Admin / Platform Admin")

    System_Ext(apic, "API-C + Global LB", "API Gateway with global load balancer. Routes to Region A (primary). Redirects to Region B on failover.")
    System_Ext(gcp, "GCP", "Cloud SQL, GKE, GCE, GCS")
    System_Ext(roc, "ROC / OneCloud", "Omnia DBaaS and other ROC services")
    System_Ext(keycloak, "Keycloak", "OIDC authentication")
    System_Ext(coredata, "Core Data API", "Tenant membership and role data")
    System_Ext(notif, "PagerDuty / Slack / Email", "Notification channels for UCP events")
    System_Ext(secrets, "Secret Manager", "TBD — global. No cross-region replication needed.")

    Container_Boundary(region_a, "Region A — Primary (serves traffic)") {

        Container_Boundary(cluster_a, "K8s Cluster — multi-AZ") {
            Container(ingress_a, "Ingress", "K8s Ingress", "TLS termination, load balancing")
            Container(api_a, "API Server", "Go / Echo", "REST API.")
            ContainerDb(platdb_a, "Platform DB", "PostgreSQL — primary", "Sessions, RBAC, audit logs, quota, notification config")
            Container(temporal_a, "Temporal Server", "Temporal OSS", "Workflow orchestration.")
            ContainerDb(tempdb_a, "Temporal DB", "PostgreSQL — primary", "Workflow state, history, visibility store")
            Container(workers_a, "Provisioning + Drift Workers", "Go / Temporal SDK, KEDA", "Executes workflows.")
            Container(crossplane_a, "Crossplane + Providers", "Crossplane, provider-gcp-*, provider-roc", "Reconciles XRs and MRs.")
            Container(eso_a, "ESO + KEDA", "ESO, KEDA", "Secret sync and worker autoscaling.")
        }
    }

    Container_Boundary(region_b, "Region B — Standby (not serving traffic)") {

        Container_Boundary(cluster_b, "K8s Cluster — multi-AZ") {
            Container(ingress_b, "Ingress", "K8s Ingress", "TLS termination, load balancing")
            Container(api_b, "API Server", "Go / Echo", "REST API — idle until failover.")
            ContainerDb(platdb_b, "Platform DB", "PostgreSQL — standby", "Read-only replica. Promoted to primary on failover.")
            Container(temporal_b, "Temporal Server", "Temporal OSS", "Workflow orchestration — idle until failover.")
            ContainerDb(tempdb_b, "Temporal DB", "PostgreSQL — standby", "Read-only replica. Promoted to primary on failover.")
            Container(workers_b, "Provisioning + Drift Workers", "Go / Temporal SDK, KEDA", "Idle until failover.")
            Container(crossplane_b, "Crossplane + Providers", "Crossplane, provider-gcp-*, provider-roc", "Idle until failover. Reconstructs XR/MR state via Observe() after failover.")
            Container(eso_b, "ESO + KEDA", "ESO, KEDA", "Secret sync and worker autoscaling.")
        }
    }

    Rel(user, apic, "HTTPS")
    Rel(apic, ingress_a, "HTTPS — normal operation")
    Rel(apic, ingress_b, "HTTPS — failover only")

    Rel(ingress_a, api_a, "HTTP")
    Rel(api_a, platdb_a, "SQL")
    Rel(api_a, secrets, "read credentials")
    Rel(api_a, temporal_a, "gRPC")
    Rel(api_a, crossplane_a, "K8s API")
    Rel(api_a, keycloak, "OIDC")
    Rel(api_a, coredata, "REST")
    Rel(temporal_a, tempdb_a, "SQL")
    Rel(workers_a, temporal_a, "gRPC")
    Rel(workers_a, crossplane_a, "K8s API")
    Rel(workers_a, notif, "HTTP")
    Rel(eso_a, secrets, "read credentials")
    Rel(crossplane_a, gcp, "REST APIs")
    Rel(crossplane_a, roc, "REST API")

    Rel(platdb_a, platdb_b, "PostgreSQL streaming replication — RPO ~1–5s")
    Rel(tempdb_a, tempdb_b, "PostgreSQL streaming replication — RPO ~1–5s")

    Rel(ingress_b, api_b, "HTTP — failover only")
    Rel(api_b, platdb_b, "SQL — failover only")
    Rel(api_b, secrets, "read credentials")
    Rel(api_b, temporal_b, "gRPC — failover only")
    Rel(api_b, crossplane_b, "K8s API — failover only")
    Rel(api_b, keycloak, "OIDC")
    Rel(api_b, coredata, "REST")
    Rel(temporal_b, tempdb_b, "SQL — failover only")
    Rel(workers_b, temporal_b, "gRPC — failover only")
    Rel(workers_b, crossplane_b, "K8s API — failover only")
    Rel(workers_b, notif, "HTTP — failover only")
    Rel(eso_b, secrets, "read credentials")
    Rel(crossplane_b, gcp, "REST APIs — failover only")
    Rel(crossplane_b, roc, "REST API — failover only")

    UpdateLayoutConfig($c4ShapeInRow="2", $c4BoundaryInRow="1")
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
C4Container
    title Alternative — Three-Cluster Separation

    Person(user, "User", "Developer / Tenant Admin / Platform Admin")

    System_Ext(apic, "API-C", "API Gateway. Inbound routing, rate limiting, correlation ID.")
    System_Ext(gcp, "GCP", "Cloud SQL, GKE, GCE, GCS")
    System_Ext(roc, "ROC / OneCloud", "Omnia DBaaS and other ROC services")
    System_Ext(keycloak, "Keycloak", "OIDC authentication")
    System_Ext(coredata, "Core Data API", "Tenant membership and role data")
    System_Ext(notif, "PagerDuty / Slack / Email", "Notification channels for UCP events")

    Container_Boundary(plat, "Platform Cluster — multi-AZ") {

        Container(ingress, "Ingress", "K8s Ingress", "TLS termination, rate limiting")
        Container(api, "API Server", "Go / Echo", "REST API. Auth, RBAC, quota, workflow submission.")
        ContainerDb(pg_plat, "Platform DB", "PostgreSQL", "Sessions, RBAC, audit logs, quota policies, notification config")
        Container(secrets, "Secret Manager", "TBD", "Cloud provider credentials and platform secrets.")
    }

    Container_Boundary(cp, "Crossplane Cluster — multi-AZ") {

        Container(crossplane, "Crossplane + Providers", "Crossplane, provider-gcp-*, provider-roc", "Reconciles XRs and MRs against cloud provider state.")
        Container(prov_w, "Provisioning Worker", "Go / Temporal SDK", "Executes provisioning workflows. In-cluster K8s API access. Connects to Temporal Server cross-cluster.")
        Container(drift_w, "Drift Worker", "Go / Temporal SDK, KEDA", "Executes drift scan and approval workflows. In-cluster K8s API access. Connects to Temporal Server cross-cluster.")
        Container(eso, "ESO", "External Secrets Operator", "Syncs credentials from Secret Manager to K8s Secrets for Crossplane.")
        Container(keda, "KEDA", "KEDA", "Scales drift workers on Temporal queue depth. Reads Temporal queue metrics cross-cluster.")
    }

    Container_Boundary(tc, "Temporal Cluster — multi-AZ") {

        Container(temporal, "Temporal Server", "Temporal OSS", "Workflow orchestration. Frontend, History, Matching — all HA.")
        ContainerDb(pg_temp, "Temporal DB", "PostgreSQL", "Temporal workflow state, history, and visibility store")
    }

    Rel(user, apic, "HTTPS")
    Rel(apic, ingress, "HTTPS")
    Rel(ingress, api, "HTTP")

    Rel(api, pg_plat, "SQL")
    Rel(api, secrets, "read credentials")
    Rel(api, temporal, "gRPC — cross-cluster via Internal LB")
    Rel(api, crossplane, "K8s API — cross-cluster kubeconfig")
    Rel(api, keycloak, "OIDC")
    Rel(api, coredata, "REST")

    Rel(temporal, pg_temp, "SQL — in-cluster")
    Rel(prov_w, temporal, "gRPC — cross-cluster")
    Rel(prov_w, crossplane, "K8s API — in-cluster")
    Rel(drift_w, temporal, "gRPC — cross-cluster")
    Rel(drift_w, crossplane, "K8s API — in-cluster")
    Rel(drift_w, pg_plat, "SQL — cross-cluster")
    Rel(drift_w, notif, "HTTP")
    Rel(keda, temporal, "gRPC — cross-cluster")
    Rel(eso, secrets, "HTTPS — cross-cluster")

    Rel(crossplane, gcp, "REST APIs")
    Rel(crossplane, roc, "REST API")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```
