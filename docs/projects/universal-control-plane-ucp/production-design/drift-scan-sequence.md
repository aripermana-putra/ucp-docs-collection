---
title: "Drift Scan — Full Sequence Diagram"
space: UCP
parent_page_id: "../production-design.md"
---

# Drift Scan — Full Sequence Diagram

Full sequence of the drift detection flow including chunked scan design,
parallel activity execution, and approval workflow.

**ScanTenantActivity input (Option A — names only):**
```go
type ScanTenantActivityInput struct {
    GVR          string   // resource type, e.g. "lbaas"
    Tenant       string   // tenant identifier, e.g. "caas"
    MRNames      []string // MR names in this chunk, e.g. ["lb-001", ..., "lb-100"]
    OpsClusterID string   // Ops cluster to query (from Platform DB routing)
}
```
DiscoverMRsActivity returns MR names only. ScanTenantActivity GETs each MR
individually for fresh `spec.forProvider` and `status.atProvider`. Temporal payload
stays small; data freshness is guaranteed.

```mermaid
sequenceDiagram
    participant Schedule as Temporal Schedule
    participant DSW as DriftScanWorkflow
    participant DRA as DiscoverMRsActivity
    participant STA as ScanTenantActivity (chunk)
    participant OpsK8s as Ops K8s API
    participant XP as Crossplane Provider
    participant Cloud as Cloud Platform
    participant DAW as DriftApprovalWorkflow
    participant PlatDB as Platform DB
    participant Notify as Notification (Slack / PD / Email)
    participant Admin as Tenant Admin
    participant FMPA as FlipManagementPolicyActivity
    participant WMRA as WaitMRReadyActivity

    Note over XP,Cloud: Background (continuous): Crossplane reconciliation loop
    XP->>Cloud: Observe() — read actual resource state
    Cloud-->>XP: actual state
    XP->>OpsK8s: Write MR.status.atProvider

    Note over Schedule: Every 1 minute (overlap policy: SKIP)
    Schedule->>DSW: Start DriftScanWorkflow

    rect rgb(220, 235, 255)
        Note over DSW,OpsK8s: Phase 1 — Discover MRs (parallel per GVR)
        par GVR: lbaas
            DSW->>DRA: DiscoverMRsActivity(gvr=lbaas)
            DRA->>OpsK8s: LIST MRs {label: drift-protection=true}
            OpsK8s-->>DRA: [(tenant=caas, mr=lb-001), (tenant=coupon, mr=lb-002), ...]
            DRA-->>DSW: MR list grouped by tenant
        and GVR: vmaas
            DSW->>DRA: DiscoverMRsActivity(gvr=vmaas)
            DRA->>OpsK8s: LIST MRs {label: drift-protection=true}
            OpsK8s-->>DRA: [(tenant=caas, mr=vm-001), ...]
            DRA-->>DSW: MR list grouped by tenant
        and GVR: dbaas / staas / ...
            DSW->>DRA: DiscoverMRsActivity(gvr=...)
            DRA-->>DSW: MR list grouped by tenant
        end
    end

    DSW->>DSW: Chunk MR list per (GVR, tenant) into batches of DRIFT_SCAN_BATCH_SIZE<br/>(default: 100, configurable via env var on drift-worker)

    rect rgb(220, 255, 220)
        Note over DSW,OpsK8s: Phase 2 — Scan all chunks in parallel
        par chunk: (lbaas, caas, MRs 1-100)
            DSW->>STA: ScanTenantActivity(lbaas, caas, [lb-001..lb-100])
            loop for each MR in chunk
                STA->>OpsK8s: GET MR → spec.forProvider + status.atProvider
                OpsK8s-->>STA: MR state
                STA->>STA: isDrifted? compare forProvider vs atProvider
                alt snooze annotation present
                    STA->>STA: skip — snoozed until timestamp
                else drift detected
                    STA->>DSW: trigger DriftApprovalWorkflow
                else no drift
                    STA->>STA: no action
                end
            end
            STA-->>DSW: chunk done
        and chunk: (lbaas, caas, MRs 101-200)
            DSW->>STA: ScanTenantActivity(lbaas, caas, [lb-101..lb-200])
            STA-->>DSW: chunk done
        and chunk: (lbaas, coupon, MRs 1-100)
            DSW->>STA: ScanTenantActivity(lbaas, coupon, [lb-001..lb-100])
            STA-->>DSW: chunk done
        and chunk: (vmaas, caas, MRs 1-100)
            DSW->>STA: ScanTenantActivity(vmaas, caas, [vm-001..vm-100])
            STA-->>DSW: chunk done
        and ... N chunks in parallel
        end
    end

    rect rgb(255, 240, 220)
        Note over DAW,Admin: Phase 3 — DriftApprovalWorkflow (fire and forget, one per drifted MR)

        DSW->>DAW: ExecuteWorkflow(DriftApprovalWorkflow, mr + xr fields)

        DAW->>PlatDB: Read NotificationConfig (tenant)
        PlatDB-->>DAW: channels (slack, pagerduty, email)

        par notify
            DAW->>Notify: POST slack webhook
        and
            DAW->>Notify: POST pagerduty events API
        and
            DAW->>Notify: SMTP email
        end

        DAW->>DAW: WaitForApprovalSignal (24h timeout)

        alt Approved
            Admin->>DAW: Signal: approve
            DAW->>FMPA: FlipManagementPolicyActivity
            FMPA->>OpsK8s: PATCH MR: managementPolicies=["Create","Observe","Update","Delete"]
            OpsK8s-->>FMPA: ok
            Note over XP: Crossplane detects full management → reconciles MR to desired state
            XP->>Cloud: Apply desired state (fix drift)
            DAW->>WMRA: WaitMRReadyActivity (35 min timeout, polls every 10s)
            loop poll every 10s until Ready=True
                WMRA->>OpsK8s: GET MR status
                OpsK8s-->>WMRA: status
            end
            WMRA-->>DAW: Ready=True
            DAW->>FMPA: FlipManagementPolicyActivity (restore read-only)
            FMPA->>OpsK8s: PATCH MR: managementPolicies=["Observe"]
            OpsK8s-->>FMPA: ok
        else Rejected
            Admin->>DAW: Signal: reject
            DAW->>DAW: NonRetryableError — MR stays in Observe mode
            Note over OpsK8s: Drift persists. Re-detected on next scan cycle<br/>unless admin adds snooze annotation to MR.
        else Timeout (24h)
            DAW->>DAW: NonRetryableError — treated same as reject
            Note over OpsK8s: Drift persists. Re-detected on next scan cycle<br/>unless admin adds snooze annotation to MR.
        end
    end
```
