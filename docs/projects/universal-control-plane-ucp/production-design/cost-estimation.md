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
| Shared DLB | Shared DLB-Hour-Load Balancer (non-CaaS) | Per LB instance | ¥0.5823 |
| Bandwidth | Shared DLB-Hour-Mega bit/Second (non-CaaS) | Per Mbps | ¥0.0932 |
| New connections | Shared DLB-Hour-New Connection/Second | Per new conn/s | ¥0.0083 |
| Concurrent connections | Shared DLB-Hour-Concurrent Connection | Per concurrent conn | ¥0.0001 |

**What UCP uses per environment:**
- 1 GSLB DNS record — single entry point for `ucp.internal.rakuten.com`
- 1 Shared DLB — gateway instance routing internal traffic to GCP via Dedicated Interconnect
- Bandwidth and connections: negligible — UCP serves internal users only, average traffic ~0.003 req/s

**Why Shared DLB, not Dedicated:**

LBaaS also offers Dedicated DLB nodes (¥30.92/hr = ~¥22,572/month per node) which
reserve an entire DLB node exclusively for one tenant. UCP uses Shared DLB for the
following reasons:

| | Shared DLB | Dedicated DLB |
|---|---|---|
| Cost | ¥0.58/hr per LB (~¥425/month) | ¥30.92/hr per node (~¥22,572/month) — ~53× more expensive |
| Traffic isolation | Multi-tenant shared infrastructure | Exclusive node, no noisy neighbour |
| Use case | Standard workloads | High-traffic or strict SLA requirements |

UCP serves internal users only at ~0.003 req/s average. There is no performance
isolation requirement, no bandwidth constraint, and no compliance requirement that
would justify a dedicated node. Shared DLB is sufficient for UCP's entire lifecycle
at the traffic volumes projected.

### Monthly calculation (730 hours/month)

| Component | Qty | ¥/month | ~/month (USD) |
|---|---|---|---|
| GSLB DNS record | 1 | ¥2,534 | ~$17 |
| Shared DLB | 1 | ¥425 | ~$3 |
| Bandwidth (<1 Mbps avg) | ~1 Mbps | ¥68 | ~$0.50 |
| Connections (negligible at UCP traffic level) | — | ~¥1 | ~$0 |
| **Total per environment** | | **¥3,028** | **~$20/month** |

### Per-environment LBaaS cost

**Option 1 (Single Cloud) and Option 2 (Multi-Region Active-Standby): Single DLB**

Only 1 active backend target per environment — no traffic distribution between multiple active backends. Single DLB is sufficient.

| Environment | GSLB | DLB(s) | Total (¥/month) | Total (~USD/month) |
|---|---|---|---|---|
| Dev | 1 DNS record | 1 shared DLB (GCP gateway) | ¥3,028 | ~$20 |
| QA | 1 DNS record | 1 shared DLB (GCP gateway) | ¥3,028 | ~$20 |
| Prod (BCP Lvl 3) | 1 DNS record | 1 shared DLB (GCP gateway) | ¥3,028 | ~$20 |
| Prod + DR (BCP Lvl 4) | 1 DNS record | 2 shared DLBs (Tokyo + Osaka gateway) | ¥3,453 | ~$23 |
| **Total LBaaS (Option 1/2, all envs)** | | | **¥9,084/month** | **~$61/month** |

**Option 3 (Multi-Cloud Active-Active): DLB on DLB for QA and Prod, single DLB for Dev**

Dev uses single DLB — development testing doesn't need accurate multi-cloud traffic weighting.
QA mirrors prod Option 3 — QA must validate the full DLB on DLB traffic path before production deployment.

| Environment | LBaaS setup | Total (¥/month) | Total (~USD/month) |
|---|---|---|---|
| Dev | Single DLB (same as Option 1/2) | ¥3,028 | ~$20 |
| QA | DLB on DLB (mirrors prod Option 3) | ¥48,953 | ~$327 |
| Prod | DLB on DLB | ¥48,953 | ~$327 |
| **Total LBaaS (Option 3, all envs)** | | **¥100,934/month** | **~$673/month** |

**Option 3 (Multi-Cloud Active-Active): DLB on DLB**

Active-active distributes live traffic between GCP and CaaS simultaneously. GSLB is DNS-based — DNS caching causes short-window traffic imbalance (e.g. a "50%:50%" split could be 10%:90% in a 1-minute window). DLB on DLB solves this with a 2-layer architecture providing 1% precision weighting, independent of DNS TTL.

```
GSLB → 1st DLB (dedicated, fine-grained weighting) → 2nd DLB-GCP (shared) → GCP LB → API servers
                                                     → 2nd DLB-CaaS (shared) → CaaS → API servers
```

The 1st DLB requires a **dedicated cluster** (DLB on DLB scope per LBaaS documentation). Minimum 2 dedicated nodes for HA within the 1st layer.

| Component | Spec | ¥/month |
|---|---|---|
| GSLB DNS record | GLB-Hour-DNS Record × 730hr | ¥2,534 |
| 1st DLB — dedicated cluster (2 nodes, HA) | 2 × Dedicated DLB-Hour-Node × 730hr | ¥45,144 |
| Reserved VIP on dedicated cluster | Dedicated DLB-Hour-Reserved VIP × 730hr | ¥425 |
| 2nd DLB — GCP gateway (shared) | Shared DLB-Hour-Load Balancer × 730hr | ¥425 |
| 2nd DLB — CaaS gateway (shared) | Shared DLB-Hour-Load Balancer × 730hr | ¥425 |
| **Total per environment (Option 3)** | | **¥48,953/month (~$327/month)** |

Option 3 LBaaS cost is ~16× higher than Options 1/2 due to the dedicated cluster requirement. Still small relative to overall infrastructure cost (~6% of Option 3 total).

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

## Dedicated Interconnect Traffic Cost

UCP traffic between Rakuten internal network and GCP crosses the **GATD Shared VPC line**
(Rakuten x GCP Dedicated Interconnect, 40 Gbps shared). Charged per GB both directions.

**Official rate (from January 2026):** **22.22 JPY/GB** — ingress and egress
Source: [Public Cloud Cost Allocation and Billing](https://confluence.rakuten-it.com/confluence/spaces/CLOUDSOL/pages/6711068530/Public+Cloud+Cost+Allocation+and+Billing)
(Section 5). Cannot be opted out. Billed 1 month after usage per Service ID.

**UCP traffic estimate:**

UCP serves internal users at ~0.003 req/s average, ~5–10KB round-trip per request.

```
All traffic to GCP (Options 1 / 2):
  0.003 req/s × 2,592,000 s/month × 7.5KB avg = ~58 GB/month
  58 GB × 22.22 JPY = ~¥1,289/month ≈ ~$9/month

Option 3 (50% user traffic to GCP + cross-cloud component calls):
  ~60 GB/month (conservative) × 22.22 JPY = ~¥1,333/month ≈ ~$9/month
```

**Conclusion:** Interconnect traffic cost is negligible for UCP at Year 1 traffic levels — same order of magnitude (~$9/month) regardless of option. Becomes relevant only at sustained high throughput or large payload sizes.

---

## Additional GCP Cost Components

The following surcharges apply on top of all GCP compute costs. Source: [Public Cloud Cost Allocation and Billing](https://confluence.rakuten-it.com/confluence/spaces/CLOUDSOL/pages/6711068530/Public+Cloud+Cost+Allocation+and+Billing).

| Surcharge | Rate | Notes |
|---|---|---|
| GCP base discount | ~29% off list price | Applied at Rakuten enterprise contract level — already reflected in per-node estimates |
| GCP Premium Support (CCoE) | ~5% of GCP spend | Allocated proportionally. Cannot be opted out. |
| CCoE Professional Support | 2.0% (FY26) / up to 3.5% (FY27) | Applied to One Cloud bill, 2 months after usage |

**Estimated surcharge impact on prod (BCP Lvl 3, ~$2,559/month GCP spend):**

| Surcharge | FY26 | FY27 |
|---|---|---|
| CCoE Premium Support (~5%) | ~$128/month | ~$128/month |
| CCoE Professional Support | ~$51/month (2%) | ~$90/month (3.5%) |
| **Total surcharges** | **~$179/month** | **~$218/month** |

These are not included in the per-environment estimates above. Add them to the final budget projection.

---

## What Is Not Included

| Item | Notes |
|---|---|
| Cloud Storage (GCS) | Used for Temporal archival and OIDC JWKS hosting. Negligible cost (<$5/month). |
| Cloud Monitoring / Logging | Depends on log volume and retention. Estimate separately. |
| CaaS hosting costs | If Ops cluster or Platform cluster runs on CaaS instead of GKE, costs are billed through OneCloud. Not modelled here. |
