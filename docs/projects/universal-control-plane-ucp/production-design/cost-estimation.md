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
| Node type (QA / Prod) | n2-custom-4-8192 (4 vCPU, 8GB) |
| Cloud SQL (Dev) | db-custom-1-3840 (1 vCPU, 3.75GB) |
| Cloud SQL (QA / Prod) | db-custom-2-7680 (2 vCPU, 7.5GB) |
| OneCloud LBaaS pricing | FY2027 unit prices (JPY, from roc_flavors_export). Converted at ¥150/USD. |
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
| LBaaS GSLB | 1 DNS record (¥3.47/hr × 730hr = ¥2,534/month) | 1 | ~$17 | ~$17 |
| LBaaS DLB | 1 shared DLB gateway to GCP (¥0.58/hr × 730hr = ¥425/month) | 1 | ~$3 | ~$3 |
| **Total** | | | | **~$694/month** |

---

### QA

| Component | Spec | Count | Unit cost/month | Total/month |
|---|---|---|---|---|
| Platform cluster nodes | n2-custom-4-8192, 3 nodes/zone × 3 zones = 9 | ~$109 | ~$981 |
| Ops cluster nodes | n2-custom-4-8192, 3 nodes/zone × 3 zones = 9 | ~$109 | ~$981 |
| GKE cluster management | — | 2 | ~$72 | ~$144 |
| Platform DB | db-custom-2-7680, sync standby, 20GB | 1 | ~$225 | ~$225 |
| Temporal DB | db-custom-2-7680, sync standby, 20GB SSD | 1 | ~$228 | ~$228 |
| LBaaS GSLB | 1 DNS record (¥3.47/hr × 730hr = ¥2,534/month) | 1 | ~$17 | ~$17 |
| LBaaS DLB | 1 shared DLB gateway to GCP (¥0.58/hr × 730hr = ¥425/month) | 1 | ~$3 | ~$3 |
| **Total** | | | | **~$2,579/month** |

QA is 1:1 with prod (Option 1) — same spec, no DR site.

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
| LBaaS GSLB | 1 DNS record (¥3.47/hr × 730hr = ¥2,534/month) | 1 | ~$17 | ~$17 |
| LBaaS DLB | 1 shared DLB gateway to GCP (¥0.58/hr × 730hr = ¥425/month) | 1 | ~$3 | ~$3 |
| **Prod total** | | | | **~$2,579/month** |

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
| LBaaS DLB (Osaka gateway) | 1 additional DLB for DR site routing (¥0.58/hr × 730hr) | 1 | ~$3 | ~$3 |
| **DR site total** | | | | **~$2,562/month** |

| | Monthly | Notes |
|---|---|---|
| Prod (primary) | ~$2,579 | Includes LBaaS GSLB + DLB |
| DR site (Osaka standby) | ~$2,562 | Same compute spec; GSLB shared, extra DLB for Osaka routing |
| **Prod + DR total** | **~$5,141/month** | ~2× prod cost — standby idle but must handle full load on failover |

---

## OneCloud LBaaS Detail

UCP uses Rakuten OneCloud LBaaS for the internal traffic entry point regardless of
which cloud option is chosen. All prices are FY2027 unit prices from
`roc_flavors_export_2027-07-27.csv`, billed per hour. Converted at ¥150/USD.

### Components UCP requires

| Component | Flavor | Unit | FY2027 Price (¥/hr) |
|---|---|---|---|
| GSLB | GLB-Hour-DNS Record | Per DNS record | ¥3.4708 |
| Shared DLB | Shared DLB-Hour-Load Balancer | Per LB instance | ¥0.5823 |
| Bandwidth | Shared DLB-Hour-Mega bit/Second | Per Mbps | ¥0.0932 |
| New connections | Shared DLB-Hour-New Connection/Second | Per new conn/s | ¥0.0083 |
| Concurrent connections | Shared DLB-Hour-Concurrent Connection | Per concurrent conn | ¥0.0001 |

**What UCP uses per environment:**
- 1 GSLB DNS record — single entry point for `ucp.internal.rakuten.com`
- 1 Shared DLB — gateway instance routing internal traffic to GCP via Dedicated Interconnect
- Bandwidth and connections: negligible — UCP serves internal users only, average traffic ~0.003 req/s

### Monthly calculation (730 hours/month)

| Component | Qty | ¥/month | ~/month (USD) |
|---|---|---|---|
| GSLB DNS record | 1 | ¥2,534 | ~$17 |
| Shared DLB | 1 | ¥425 | ~$3 |
| Bandwidth (<1 Mbps avg) | ~1 Mbps | ¥68 | ~$0.50 |
| Connections (negligible at UCP traffic level) | — | ~¥1 | ~$0 |
| **Total per environment** | | **¥3,028** | **~$20/month** |

### Per-environment LBaaS cost

| Environment | GSLB | DLB(s) | Total (¥/month) | Total (~USD/month) |
|---|---|---|---|---|
| Dev | 1 DNS record | 1 DLB (GCP gateway) | ¥3,028 | ~$20 |
| QA | 1 DNS record | 1 DLB (GCP gateway) | ¥3,028 | ~$20 |
| Prod (BCP Lvl 3) | 1 DNS record | 1 DLB (GCP gateway) | ¥3,028 | ~$20 |
| Prod + DR (BCP Lvl 4) | 1 DNS record | 2 DLBs (Tokyo + Osaka gateway) | ¥3,453 | ~$23 |

> **Note on Professional Support fee:** LBaaS has a platform support fee of
> ¥12,929/month. This is a shared organisational cost — confirm with LBaaS team
> whether UCP would be charged this as a new tenant or whether it is absorbed
> into existing contracts.

### BCP Level 3 vs 4 difference (LBaaS only)

For Option 2 (multi-region DR), the GSLB is shared — it already routes to multiple
backends. The only additional cost is 1 extra DLB instance in Osaka as the DR
gateway: +¥425/month (+~$3/month). The DR cost impact from LBaaS is negligible.

---

## Total Across All Environments

| Environment | Monthly | Annual | Notes |
|---|---|---|---|
| Dev | ~$694 | ~$8,328 | GCP + LBaaS. Covers combined Dev + integration test load |
| QA | ~$2,579 | ~$30,948 | GCP + LBaaS. 1:1 with prod, no DR site |
| Prod (BCP Lvl 3, multi-AZ only) | ~$2,579 | ~$30,948 | GCP + LBaaS |
| **Total (BCP Lvl 3)** | **~$5,852/month** | **~$70,224/year** | |
| | | | |
| + DR site (BCP Lvl 4, multi-region) | +~$2,562 | +~$30,744 | GCP Osaka + extra DLB |
| **Total (BCP Lvl 4)** | **~$8,414/month** | **~$100,968/year** | |

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
| LBaaS | No discount (OneCloud pricing) | ~$20 |
| **Adjusted prod total** | | **~$1,762/month** |

---

## What Is Not Included

| Item | Notes |
|---|---|
| Egress / network traffic | Cross-region egress from Dedicated Interconnect calls. Negligible at Year 1 scale (~$0.01/GB). |
| Cloud Storage (GCS) | Used for Temporal archival and OIDC JWKS hosting. Negligible cost (<$5/month). |
| Cloud Monitoring / Logging | Depends on log volume and retention. Estimate separately. |
| CaaS hosting costs | If Ops cluster or Platform cluster runs on CaaS instead of GKE, costs are billed through OneCloud. Not modelled here. |
| Dedicated Interconnect | Typically a shared organisational cost, not UCP-specific. |
