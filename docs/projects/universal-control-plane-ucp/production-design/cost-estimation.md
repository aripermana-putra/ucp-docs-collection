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
| GCP costs | Actual from GCP Pricing Calculator, effective 2026-08-28. On-demand, no discounts. |
| Region | asia-northeast1 (Tokyo) for primary; asia-northeast2 (Osaka) for DR site |
| Node type (Dev) | e2-custom-2-4096 (2 vCPU, 4GB), Zonal cluster |
| Node type (QA / Prod) | n2-custom-4-8192 (4 vCPU, 8GB), Regional cluster |
| Cloud SQL (Dev) | Zonal - Small instance |
| Cloud SQL (QA / Prod) | Regional (sync standby) |
| OneCloud LBaaS pricing | FY2027 unit prices (JPY, from roc_flavors_export). Converted at ¥150/USD. |
| Option 2 (approved) | Prod primary (Tokyo) × 2 — same spec deployed in Osaka as active-standby DR |

---

## Per-Environment Estimate

### Dev

GCP costs from Calculator (2026-08-28). Zonal cluster (no HA required for Dev).

| Component | GCP Calculator | Total/month |
|---|---|---|
| Platform Cluster (Zonal, e2-custom-2-4096, 6 nodes) | Mgmt $73 + PD $39 + Core $128.89 + RAM $34.39 | $275.28 |
| Ops Cluster (Zonal, e2-custom-2-4096, 6 nodes) | Same | $275.28 |
| Platform DB (Zonal Small, 10GB) | $33.58 + $2.21 | $35.79 |
| Temporal DB (Zonal Small, 10GB) | $33.58 + $2.21 | $35.79 |
| **GCP subtotal** | | **$622.14** |
| LBaaS GSLB + DLB | ¥3,028/month ÷ ¥150 | ~$20 |
| **Total** | | **~$642/month** |

---

### QA

GCP costs from Calculator (2026-08-28). Regional cluster, same spec as prod primary.

| Component | GCP Calculator | Total/month |
|---|---|---|
| Platform Cluster (Regional, n2-custom-4-8192, 9 nodes) | Mgmt $73 + PD $117 + Core $1,120.81 + RAM $299.06 | $1,609.87 |
| Ops Cluster (Regional, n2-custom-4-8192, 9 nodes) | Same | $1,609.87 |
| Platform DB (Regional sync standby, 20GB) | vCPU $156.80 + RAM $53.14 + Storage $44.20 | $254.14 |
| Temporal DB (Regional sync standby, 20GB) | Same | $254.14 |
| GCP Load Balancer | | $19.85 |
| **GCP subtotal** | | **$3,747.87** |
| LBaaS GSLB + DLB | ¥3,028/month ÷ ¥150 | ~$20 |
| **Total** | | **~$3,768/month** |

QA mirrors prod primary site spec — same spec as prod Tokyo, no DR site.

---

### Prod (BCP Level 4 — Multi-Region, Active-Standby) ← **Confirmed**

Primary site in asia-northeast1 (Tokyo) × same spec deployed in asia-northeast2 (Osaka)
as active-standby DR. Option 2 approved 2026-08-28.

**Per site (Tokyo or Osaka) — from GCP Calculator (2026-08-28):**

| Component | GCP Calculator | Total/month |
|---|---|---|
| Platform Cluster (Regional, n2-custom-4-8192, 9 nodes) | Mgmt $73 + PD $117 + Core $1,120.81 + RAM $299.06 | $1,609.87 |
| Ops Cluster (Regional, n2-custom-4-8192, 9 nodes) | Same | $1,609.87 |
| Platform DB (Regional sync standby, 20GB) | vCPU $156.80 + RAM $53.14 + Storage $44.20 | $254.14 |
| Temporal DB (Regional sync standby, 20GB) | Same | $254.14 |
| GCP Load Balancer | | $19.85 |
| **GCP per site** | | **$3,747.87** |

| | GCP | LBaaS | Monthly total |
|---|---|---|---|
| Prod primary (Tokyo) | $3,747.87 | ~$20 (GSLB + DLB) | ~$3,768 |
| DR site (Osaka standby) | $3,747.87 | ~$3 (extra DLB only) | ~$3,751 |
| **Prod + DR total** | **$7,495.74** | **~$23** | **~$7,519/month** |

~2× prod cost — standby is idle but must handle full load on failover.

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
| Dev | ~$642 | ~$7,704 | GCP $622.14 + LBaaS ~$20 |
| QA | ~$3,768 | ~$45,216 | GCP $3,747.87 + LBaaS ~$20 |
| **Dev + QA subtotal** | **~$4,500 (GCP) + ¥6,056 (LBaaS)** | | GCP from Calculator 2026-08-28 |
| | | | |
| Prod primary (Tokyo) | ~$3,768 | ~$45,216 | GCP $3,747.87 + LBaaS ¥3,028 |
| Prod DR site (Osaka) | ~$3,751 | ~$45,012 | GCP $3,747.87 + LBaaS ¥425 (extra DLB only) |
| **Prod total (BCP Lvl 4) ← confirmed** | **~$7,500 (GCP) + ¥3,453 (LBaaS)** | | Option 2 approved 2026-08-28 |

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
