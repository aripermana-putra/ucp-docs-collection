---
title: "Cost Estimation"
space: UCP
parent_page_id: "../production-design.md"
---

# Cost Estimation

Monthly infrastructure cost estimates for UCP running on GKE and Cloud SQL.
Specs are sourced from [Component Sizing](component-sizing.md).

> **Important:** All prices are approximate on-demand rates for **asia-northeast1
> (Tokyo)** as of 2026. Verify against the
> [GCP Pricing Calculator](https://cloud.google.com/products/calculator) before
> committing. Actual costs will differ based on sustained use discounts,
> committed use contracts, and storage growth.

---

## Assumptions

| Parameter | Value |
|---|---|
| Region | asia-northeast1 (Tokyo) |
| Pricing model | On-demand (no discounts applied) |
| GKE mode | Standard (not Autopilot) |
| GKE cluster management fee | ~$72/month per cluster |
| Node type (Dev/QA) | e2-standard-2 (2 vCPU, 8GB) — closest standard to 2 vCPU/4GB spec |
| Node type (Staging/Prod) | n2-standard-4 (4 vCPU, 16GB) — closest standard to 4 vCPU/8GB spec |
| Cloud SQL (Dev/QA) | db-g1-small (shared-core, ~0.5 vCPU, 1.7GB) |
| Cloud SQL (Staging) | db-custom-1-3840 (1 vCPU, 3.75GB) |
| Cloud SQL (Prod) | db-custom-2-7680 (2 vCPU, 7.5GB) |
| Cloud SQL storage | SSD ~$0.17/GB/month |
| Cloud SQL HA | ~1.5× single instance cost (standby replica) |

---

## Per-Environment Estimate

### Dev

| Component | Spec | Count | Unit cost/month | Total/month |
|---|---|---|---|---|
| Platform cluster nodes | e2-standard-2 | 1 | ~$49 | ~$49 |
| Ops cluster nodes | e2-standard-2 | 1 | ~$49 | ~$49 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | db-g1-small, single, 5GB | 1 | ~$26 | ~$26 |
| Temporal DB | db-g1-small, single, 5GB SSD | 1 | ~$26 | ~$26 |
| **Total** | | | | **~$294/month** |

---

### QA

| Component | Spec | Count | Unit cost/month | Total/month |
|---|---|---|---|---|
| Platform cluster nodes | e2-standard-2 | 2 | ~$49 | ~$98 |
| Ops cluster nodes | e2-standard-2 | 2 | ~$49 | ~$98 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | db-g1-small, single, 10GB | 1 | ~$28 | ~$28 |
| Temporal DB | db-g1-small, single, 10GB SSD | 1 | ~$28 | ~$28 |
| **Total** | | | | **~$396/month** |

---

### Staging

| Component | Spec | Count | Unit cost/month | Total/month |
|---|---|---|---|---|
| Platform cluster nodes | n2-standard-4 | 2 | ~$138 | ~$276 |
| Ops cluster nodes | n2-standard-4 | 2 | ~$138 | ~$276 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | db-custom-1-3840, async standby, 10GB | 1 | ~$83 | ~$83 |
| Temporal DB | db-custom-1-3840, async standby, 10GB SSD | 1 | ~$83 | ~$83 |
| **Total** | | | | **~$862/month** |

---

### Prod (BCP Level 3 — Multi-AZ)

Primary site in asia-northeast1 (Tokyo), nodes spread across 3 AZs. Cloud SQL sync
standby provides zone-level HA. This is the baseline production deployment.

| Component | Spec | Count | Unit cost/month | Total/month |
|---|---|---|---|---|
| Platform cluster nodes | n2-standard-4 | 3 | ~$138 | ~$414 |
| Ops cluster nodes | n2-standard-4 | 3 | ~$138 | ~$414 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | db-custom-2-7680, sync standby, 20GB | 1 | ~$225 | ~$225 |
| Temporal DB | db-custom-2-7680, sync standby, 20GB SSD | 1 | ~$228 | ~$228 |
| **Prod total** | | | | **~$1,425/month** |

### Prod + DR Site (BCP Level 4 — Multi-Region, Active-Standby)

DR site in asia-northeast2 (Osaka). Active-standby: the DR site runs at full spec and
is ready to absorb 100% of traffic on failover, but serves no live traffic under normal
conditions. Platform DB and Temporal DB use cross-region read replicas that are promoted
to primary on failover. Temporal requires MCR (Multi-Cluster Replication) or manual
failover procedure.

**DR site (asia-northeast2):**

| Component | Spec | Count | Unit cost/month | Total/month |
|---|---|---|---|---|
| Platform cluster nodes | n2-standard-4 | 3 | ~$138 | ~$414 |
| Ops cluster nodes | n2-standard-4 | 3 | ~$138 | ~$414 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | Cross-region read replica, 20GB | 1 | ~$225 | ~$225 |
| Temporal DB | Cross-region read replica, 20GB SSD | 1 | ~$228 | ~$228 |
| **DR site total** | | | | **~$1,425/month** |

| | Monthly | Notes |
|---|---|---|
| Prod (primary) | ~$1,425 | |
| DR site (standby) | ~$1,425 | Same spec — must handle full prod load on failover |
| **Prod + DR total** | **~$2,850/month** | ~2× prod cost — standby is idle but must be ready |

---

## Total Across All Environments

| Environment | Monthly | Annual | Notes |
|---|---|---|---|
| Dev | ~$294 | ~$3,528 | |
| QA | ~$396 | ~$4,752 | |
| Staging | ~$862 | ~$10,344 | |
| Prod (BCP Lvl 3, multi-AZ only) | ~$1,425 | ~$17,100 | |
| **Total (BCP Lvl 3)** | **~$2,977/month** | **~$35,724/year** | |
| | | | |
| + DR site (BCP Lvl 4, multi-region) | +~$1,425 | +~$17,100 | Active-standby in Osaka |
| **Total (BCP Lvl 4)** | **~$4,402/month** | **~$52,824/year** | |

---

## Cost Optimization Opportunities

| Option | Applicable to | Saving |
|---|---|---|
| **Committed use (1-year)** | Prod, Staging | ~37% off compute |
| **Committed use (3-year)** | Prod | ~55% off compute |
| **Spot VMs** | Dev, QA | ~60–70% off node cost. Acceptable for non-prod where disruption is tolerable. |
| **GKE Autopilot** | Dev, QA | Pay per pod resource, not per node. Potentially cheaper for low-utilization environments with many idle nodes. |
| **Shared VPC / project** | Dev, QA | GKE cluster management fee waived for the first zonal cluster per project. Running Dev and QA in the same project saves ~$72/month. |

**Estimated prod cost with 1-year committed use:**

| Component | Saving | Adjusted monthly |
|---|---|---|
| Cluster nodes (6 × n2-standard-4) | 37% off ~$828 | ~$522 |
| GKE management | No discount | ~$144 |
| Cloud SQL | ~20% off ~$453 | ~$362 |
| **Adjusted prod total** | | **~$1,028/month** |

---

## What Is Not Included

| Item | Notes |
|---|---|
| Egress / network traffic | Cross-region egress from Dedicated Interconnect calls. Negligible at Year 1 scale (~$0.01/GB). |
| Cloud Storage (GCS) | Used for Temporal archival and OIDC JWKS hosting. Negligible cost (<$5/month). |
| Cloud Monitoring / Logging | Depends on log volume and retention. Estimate separately. |
| CaaS hosting costs | If Ops cluster or Platform cluster runs on CaaS instead of GKE, costs are billed through OneCloud. Not modelled here. |
| Dedicated Interconnect | Typically a shared organisational cost, not UCP-specific. |
