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
> committing. Actual costs will differ based on corporate price, sustained use discounts,
> committed use contracts, and storage growth.

---

## Assumptions

| Parameter | Value |
|---|---|
| Region | asia-northeast1 (Tokyo) |
| Pricing model | On-demand (no discounts applied) |
| GKE mode | Standard (not Autopilot) |
| GKE cluster management fee | ~$72/month per cluster |
| Node type (Dev) | e2-custom-2-4096 (2 vCPU, 4GB) |
| Node type (QA/Prod) | n2-custom-4-8192 (4 vCPU, 8GB) |
| Node type (Staging/Prod) | n2-custom-4-8192 (4 vCPU, 8GB) |
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
| Platform cluster nodes | e2-custom-2-4096, 2 nodes/zone × 3 zones = 6 | ~$35 | ~$210 |
| Ops cluster nodes | e2-custom-2-4096, 2 nodes/zone × 3 zones = 6 | ~$35 | ~$210 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | db-custom-1-3840, single, 10GB | 1 | ~$55 | ~$55 |
| Temporal DB | db-custom-1-3840, single, 10GB SSD | 1 | ~$55 | ~$55 |
| **Total** | | | | **~$674/month** |

---

### QA

| Component | Spec | Count | Unit cost/month | Total/month |
|---|---|---|---|---|
| Platform cluster nodes | n2-custom-4-8192, 3 nodes/zone × 3 zones = 9 | ~$109 | ~$981 |
| Ops cluster nodes | n2-custom-4-8192, 3 nodes/zone × 3 zones = 9 | ~$109 | ~$981 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | db-custom-2-7680, sync standby, 20GB | 1 | ~$225 | ~$225 |
| Temporal DB | db-custom-2-7680, sync standby, 20GB SSD | 1 | ~$228 | ~$228 |
| **Total** | | | | **~$2,559/month** |

QA is 1:1 with prod (Option 1) — same spec, no DR site.

---

### Staging

| Component | Spec | Count | Unit cost/month | Total/month |
|---|---|---|---|---|
| Platform cluster nodes | n2-custom-4-8192, 2 nodes/zone × 3 zones = 6 | ~$109 | ~$654 |
| Ops cluster nodes | n2-custom-4-8192, 2 nodes/zone × 3 zones = 6 | ~$109 | ~$654 |
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
| Platform cluster nodes | n2-custom-4-8192, 3 nodes/zone × 3 zones = 9 | ~$109 | ~$981 |
| Ops cluster nodes | n2-custom-4-8192, 3 nodes/zone × 3 zones = 9 | ~$109 | ~$981 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | db-custom-2-7680, sync standby, 20GB | 1 | ~$225 | ~$225 |
| Temporal DB | db-custom-2-7680, sync standby, 20GB SSD | 1 | ~$228 | ~$228 |
| **Prod total** | | | | **~$2,559/month** |

### Prod + DR Site (BCP Level 4 — Multi-Region, Active-Standby)

DR site in asia-northeast2 (Osaka). Active-standby: the DR site runs at full spec and
is ready to absorb 100% of traffic on failover, but serves no live traffic under normal
conditions. Platform DB and Temporal DB use cross-region read replicas that are promoted
to primary on failover. Temporal requires MCR (Multi-Cluster Replication) or manual
failover procedure.

**DR site (asia-northeast2):**

| Component | Spec | Count | Unit cost/month | Total/month |
|---|---|---|---|---|
| Platform cluster nodes | n2-custom-4-8192, 3 nodes/zone × 3 zones = 9 | ~$109 | ~$981 |
| Ops cluster nodes | n2-custom-4-8192, 3 nodes/zone × 3 zones = 9 | ~$109 | ~$981 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | Cross-region read replica, 20GB | 1 | ~$225 | ~$225 |
| Temporal DB | Cross-region read replica, 20GB SSD | 1 | ~$228 | ~$228 |
| **DR site total** | | | | **~$2,559/month** |

| | Monthly | Notes |
|---|---|---|
| Prod (primary) | ~$1,425 | |
| DR site (standby) | ~$1,425 | Same spec — must handle full prod load on failover |
| **Prod + DR total** | **~$2,850/month** | ~2× prod cost — standby is idle but must be ready |

---

## Total Across All Environments

| Environment | Monthly | Annual | Notes |
|---|---|---|---|
| Dev | ~$674 | ~$8,088 | Covers combined Dev + QA load |
| QA | ~$2,559 | ~$30,708 | 1:1 with prod, no DR site |
| Prod (BCP Lvl 3, multi-AZ only) | ~$2,559 | ~$30,708 | |
| **Total (BCP Lvl 3)** | **~$5,792/month** | **~$69,504/year** | |
| | | | |
| + DR site (BCP Lvl 4, multi-region) | +~$2,559 | +~$30,708 | Active-standby in Osaka |
| **Total (BCP Lvl 4)** | **~$8,351/month** | **~$100,212/year** | |

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
| Cluster nodes (18 × n2-custom-4-8192) | 37% off ~$1,962 | ~$1,236 |
| GKE management | No discount | ~$144 |
| Cloud SQL | ~20% off ~$453 | ~$362 |
| **Adjusted prod total** | | **~$1,742/month** |

---

## What Is Not Included

| Item | Notes |
|---|---|
| Egress / network traffic | Cross-region egress from Dedicated Interconnect calls. Negligible at Year 1 scale (~$0.01/GB). |
| Cloud Storage (GCS) | Used for Temporal archival and OIDC JWKS hosting. Negligible cost (<$5/month). |
| Cloud Monitoring / Logging | Depends on log volume and retention. Estimate separately. |
| CaaS hosting costs | If Ops cluster or Platform cluster runs on CaaS instead of GKE, costs are billed through OneCloud. Not modelled here. |
| Dedicated Interconnect | Typically a shared organisational cost, not UCP-specific. |
