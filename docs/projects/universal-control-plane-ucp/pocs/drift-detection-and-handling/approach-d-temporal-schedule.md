---
title: "Approach D — Temporal Schedule"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Approach D — Temporal Schedule

**Branch:** `feature/drift-poc-approach-d-temporal-schedule`
**Trigger mechanism:** Temporal cron schedule runs a drift scan workflow every minute

---

## How It Works

A Temporal `Schedule` triggers `DriftScanWorkflow` every minute. The workflow runs
`ScanDriftActivity`, which lists all XRs with the drift-protection label across all
configured GVRs, and fires `DriftApprovalWorkflow` for any that are drifted.

No external binary. No new Kubernetes Deployment. All logic lives inside Temporal.

```
Temporal Schedule (every 1 min)
    └── DriftScanWorkflow
            └── ScanDriftActivity
                  for each configured GVR:
                    list XRs with platform.io/drift-protection=true
                    for each XR with Synced=False:
                      ExecuteWorkflow(DriftApprovalWorkflow, ...)
```

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│ Temporal                                          │
│                                                  │
│  Schedule: drift-scan (every 1 min)              │
│    └── DriftScanWorkflow                         │
│          └── ScanDriftActivity                   │
│                │ K8s API (list XRs per GVR)      │
│                for each drifted XR:              │
│                  ExecuteWorkflow(                │
│                    DriftApprovalWorkflow,         │
│                    id=drift-approval-<ns>-<name> │
│                  )                               │
└──────────────────────────────────────────────────┘
                   │
                   │  K8s API (list)
                   ▼
┌──────────────────────────────────────────────────┐
│ Kubernetes                                        │
│  XDatabase, XComputeInstance, etc.               │
│  with label platform.io/drift-protection=true   │
└──────────────────────────────────────────────────┘
```

---

## Multi-Resource Support

The GVR list is passed into `DriftScanWorkflow` as part of the Schedule input. Adding a
new resource type = update the Schedule — no code change, no redeployment:

```bash
temporal schedule update \
  --schedule-id "drift-scan" \
  --input '{"gvrs": ["...", "platform.example.io/v1alpha1/xloadbalancers"]}'
```

---

## New Files

No new binaries. No new Deployments.

```
backend/temporal-worker/internal/workflows/drift_approval.go  NEW (shared)
backend/temporal-worker/internal/workflows/drift_scan.go      NEW
backend/temporal-worker/internal/activities/drift.go          NEW (shared)
backend/temporal-worker/internal/activities/drift_scan.go     NEW
backend/temporal-worker/cmd/worker/main.go                    MODIFY
k8s/temporal-worker/serviceaccount.yaml                       MODIFY (RBAC)
scripts/setup-drift-scan-schedule.sh                          NEW
crossplane/xrd/* (4 files)                                    MODIFY
crossplane/composition/* (4 files)                            MODIFY
```

---

## DriftScanWorkflow

```go
type DriftScanInput struct {
    GVRs []string `json:"gvrs"` // "group/version/resource" per entry
}

type DriftScanOutput struct {
    ScannedResources int `json:"scannedResources"`
    DriftedResources int `json:"driftedResources"`
    WorkflowsFired   int `json:"workflowsFired"`
}

func DriftScanWorkflow(ctx workflow.Context, in DriftScanInput) (DriftScanOutput, error) {
    actCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        StartToCloseTimeout: 2 * time.Minute,
        RetryPolicy: &temporal.RetryPolicy{MaximumAttempts: 2},
    })
    var out DriftScanOutput
    err := workflow.ExecuteActivity(actCtx, activities.ScanDriftActivity, in).Get(ctx, &out)
    return out, err
}
```

---

## ScanDriftActivity

```go
func ScanDriftActivity(ctx context.Context, in workflows.DriftScanInput) (workflows.DriftScanOutput, error) {
    dc, _ := k8s.NewDynamicClient()
    tc    := getTemporalClient()
    var out workflows.DriftScanOutput

    for _, gvrStr := range in.GVRs {
        gvr, _ := parseGVRString(gvrStr)
        list, _ := dc.Resource(gvr).Namespace("").List(ctx, metav1.ListOptions{
            LabelSelector: "platform.io/drift-protection=true",
        })
        for _, item := range list.Items {
            out.ScannedResources++
            if !isDrifted(&item) { continue }
            out.DriftedResources++

            driftIn  := buildDriftInput(&item, gvr, inferKindFromGVR(gvr))
            wfID     := fmt.Sprintf("drift-approval-%s-%s-%s",
                driftIn.Namespace, strings.ToLower(driftIn.XRKind), driftIn.XRName)

            _, err := tc.ExecuteWorkflow(ctx, client.StartWorkflowOptions{
                ID: wfID, TaskQueue: "db-provisioning",
            }, "DriftApprovalWorkflow", driftIn)

            if isAlreadyStarted(err) { continue } // dedup
            if err != nil { log.Printf("fire workflow error: %v", err); continue }
            out.WorkflowsFired++
        }
    }
    return out, nil
}
```

---

## Schedule Registration

```bash
# scripts/setup-drift-scan-schedule.sh
temporal schedule create \
  --address "${TEMPORAL_ADDRESS:-localhost:7233}" \
  --schedule-id "drift-scan" \
  --workflow-type "DriftScanWorkflow" \
  --task-queue "db-provisioning" \
  --overlap-policy Skip \
  --input '{
    "gvrs": [
      "platform.example.io/v1alpha1/xdatabases",
      "platform.example.io/v1alpha1/xcomputeinstances",
      "platform.example.io/v1alpha1/xkubernetesclusters",
      "platform.example.io/v1alpha1/xobjectstorages"
    ]
  }' \
  --cron "* * * * *"
```

**Overlap policy `Skip`:** if a scan takes longer than 1 minute (unlikely), the next
trigger is skipped rather than stacking concurrent scans.

---

## Advantages Over A/C

| Dimension | A/C (external binary) | D (Temporal Schedule) |
|-----------|-----------------------|-----------------------|
| New K8s Deployments | 1 (drift-watcher) | 0 |
| Scan observability | Pod logs | Temporal UI — full history per scan run with structured output |
| Alerting on scan failure | Log-based alerting | Temporal workflow failure alerting (already in place) |
| Update resource list | ConfigMap update + pod restart (C) | `temporal schedule update` — no deploy |
| Audit trail | Logs only | Full Temporal workflow history |

---

## Failure Modes

| Failure | Effect | Recovery |
|---------|--------|----------|
| Temporal worker crashes | Schedule fires; next run picks up all drifted resources | Auto-recover on restart |
| Temporal server down | Schedule pauses | Scans resume when server recovers; all drift detected on next run |
| K8s API unavailable during scan | `ScanDriftActivity` fails; workflow retries | Schedule fires again next minute |
| Scan takes >1 min | Next trigger skipped (`Skip` policy) | No issue for normal resource counts |

---

## Key Disadvantage vs A/C

If Temporal is down, drift scanning stops entirely. In A/C, the watcher runs independently
of Temporal (only the workflow start call fails, and the watcher retries on the next poll).

This is the main trade-off: Approach D has fewer components but higher coupling to Temporal
availability for drift *detection* (not just handling).

---

## Pros and Cons

### Pros
- **No new K8s Deployment** — fewest components to operate; all drift logic lives inside Temporal
- **Richest observability** — every scan run is a Temporal workflow with structured output (`scanned` / `drifted` / `fired`) visible in the Temporal UI
- **Full audit trail** — complete scan history retained in Temporal, not just pod logs
- **Easiest resource type extension** — adding a new GVR only requires `temporal schedule update`; no deployment or restart
- **Reuses existing alerting** — scan failures surface as Temporal workflow failures, picked up by any existing Temporal alerting

### Cons
- **Fully coupled to Temporal availability** — if Temporal is down, drift scanning stops entirely; in A/C the watcher runs independently while only the workflow start call fails
- **Up to 1 min scan latency** — same as A's poll interval, worse than B/C event-driven approaches
- **Unusual pattern** — `ScanDriftActivity` calls back into Temporal to start `DriftApprovalWorkflow`; requires injecting a Temporal client into the activity
- **Scan history accumulates** — 1 `DriftScanWorkflow` entry per minute; requires a Temporal workflow retention policy to avoid unbounded storage growth
- **Potential rate limiting at scale** — if many resources drift simultaneously, all `DriftApprovalWorkflow` starts happen within one activity execution; may hit Temporal API rate limits with very large resource counts
