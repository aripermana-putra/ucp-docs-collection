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
`ScanDriftActivity`, which lists all MRs with the drift-protection label across all
configured MR GVRs, diffs `spec.forProvider` against `status.atProvider`, and fires
`DriftApprovalWorkflow` for any that are drifted. The ownerReferences on each drifted
MR are resolved to identify the XR for the workflow key.

No external binary. No new Kubernetes Deployment. All logic lives inside Temporal.

```
Temporal Schedule (every 1 min)
    └── DriftScanWorkflow
            └── ScanDriftActivity
                  for each configured MR GVR:
                    list MRs with platform.io/drift-protection=true
                    for each MR:
                      drifted, detail := isDrifted(mr)  // forProvider vs atProvider
                      if drifted:
                        resolve ownerRef → XR name
                        ExecuteWorkflow(DriftApprovalWorkflow, mrFields + xrFields)
```

---

## Where `forProvider` and `atProvider` Come From

`ScanDriftActivity` runs inside the **temporal-worker pod** (already deployed in the cluster).
It **never calls GCP or Omnia directly** — it reads all state from the **Kubernetes API server**
using `k8s.io/client-go/dynamic`, the same client library used by Approaches A and C.

```
GCP Console (manual change to CloudSQL tier)
        │
        │  GCP REST API
        ▼
provider-gcp pod (running in crossplane-system)
  managementPolicies=["Observe"] → Observe() called on every poll (~10 min)
  → calls GCP REST API, reads actual current state
  → PATCH MR.status.atProvider via K8s API server
        │
        │  K8s API server writes to etcd
        ▼
etcd (via K8s API server)
  MR.spec.forProvider   = desired state (what Crossplane declared)
  MR.status.atProvider  = actual state  (what GCP currently has)
        │
        │  Temporal Schedule fires every 1 min
        ▼
ScanDriftActivity (running inside temporal-worker pod)
  dynamic client: LIST /apis/sql.gcp.upbound.io/v1beta2/databaseinstances
  reads forProvider + atProvider from K8s API response
  diff → if mismatch → resolve ownerRef → ExecuteWorkflow
```

The temporal-worker pod already has a `ServiceAccount` with K8s API access (for existing
`ApplyYAMLActivity`, `WaitDatabaseClaimReadyActivity`, etc.). `ScanDriftActivity` reuses
the same in-cluster kubeconfig — no additional auth setup required.

---

## Architecture

```
GCP / Omnia                provider-gcp / provider-roc pod
(actual state)  ────────►  Observe() every ~10 min
                           writes status.atProvider
                                   │
                                   │  PATCH /status (K8s API)
                                   ▼
                           K8s API server (etcd)
                           ┌──────────────────────────────────┐
                           │ DatabaseInstance                 │
                           │  spec.forProvider:               │
                           │    settings.tier: db-n1-std-2    │
                           │  status.atProvider:              │
                           │    settings.tier: db-n1-std-4 ←drift│
                           └──────────────────────────────────┘
                                   │
                                   │  LIST /apis/sql.gcp.upbound.io/...
                                   │  (dynamic client inside temporal-worker pod)
                                   ▼
                  Temporal Schedule (every 1 min)
                  └── DriftScanWorkflow
                        └── ScanDriftActivity
                              diff forProvider vs atProvider
                              ownerRef → XDatabase/my-postgres
                                   │
                                   │  Temporal Go SDK (gRPC to Temporal server)
                                   ▼
                           DriftApprovalWorkflow
                           (see Shared Design)
```

---

## Multi-Resource Support

The MR GVR list is passed into `DriftScanWorkflow` as part of the Schedule input. Adding a
new resource type = update the Schedule — no code change, no redeployment:

```bash
temporal schedule update \
  --schedule-id "drift-scan" \
  --input '{"gvrs": ["...", "networking.gcp.upbound.io/v1beta1/globaladdresses"]}'
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
    GVRs []string `json:"gvrs"` // MR GVRs: "group/version/resource" per entry
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
            drifted, detail := isDrifted(&item)
            if !drifted { continue }
            out.DriftedResources++

            driftIn := buildDriftInput(&item, gvr, detail)
            wfID    := fmt.Sprintf("drift-approval-%s-%s-%s",
                driftIn.XRNamespace, strings.ToLower(driftIn.XRKind), driftIn.XRName)

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

// isDrifted compares spec.forProvider against status.atProvider.
// Fields present in atProvider but absent in forProvider are ignored (computed fields).
func isDrifted(obj *unstructured.Unstructured) (bool, string) {
    forProvider, _, _ := unstructured.NestedMap(obj.Object, "spec", "forProvider")
    atProvider, _, _  := unstructured.NestedMap(obj.Object, "status", "atProvider")
    if len(atProvider) == 0 {
        return false, "" // Observe() has not run yet
    }
    diffs := diffMaps("", forProvider, atProvider)
    if len(diffs) == 0 {
        return false, ""
    }
    return true, strings.Join(diffs, "; ")
}

func diffMaps(prefix string, desired, observed map[string]interface{}) []string {
    var out []string
    for k, dv := range desired {
        path := k
        if prefix != "" {
            path = prefix + "." + k
        }
        ov := observed[k]
        if dMap, ok := dv.(map[string]interface{}); ok {
            if oMap, ok := ov.(map[string]interface{}); ok {
                out = append(out, diffMaps(path, dMap, oMap)...)
            } else {
                out = append(out, fmt.Sprintf("%s: differs", path))
            }
        } else if fmt.Sprintf("%v", dv) != fmt.Sprintf("%v", ov) {
            out = append(out, fmt.Sprintf("%s: %v → %v", path, dv, ov))
        }
    }
    return out
}

// buildDriftInput populates both MR and XR fields.
// XR fields are resolved from the MR's ownerReferences (controller owner).
func buildDriftInput(mr *unstructured.Unstructured, mrGVR schema.GroupVersionResource, detail string) DriftApprovalInput {
    xrName, xrKind, xrAPIVersion := resolveControllerOwner(mr)
    xrGroup, xrVersion := splitAPIVersion(xrAPIVersion)
    return DriftApprovalInput{
        MRGroup:     mrGVR.Group,
        MRVersion:   mrGVR.Version,
        MRResource:  mrGVR.Resource,
        MRName:      mr.GetName(),
        MRNamespace: mr.GetNamespace(),
        XRGroup:     xrGroup,
        XRVersion:   xrVersion,
        XRResource:  pluralFromKind(xrKind),
        XRKind:      xrKind,
        XRName:      xrName,
        XRNamespace: mr.GetNamespace(),
        DetectedAt:  time.Now().UTC().Format(time.RFC3339),
        DriftDetail: detail,
    }
}

func resolveControllerOwner(obj *unstructured.Unstructured) (name, kind, apiVersion string) {
    for _, ref := range obj.GetOwnerReferences() {
        if ref.Controller != nil && *ref.Controller {
            return ref.Name, ref.Kind, ref.APIVersion
        }
    }
    return "", "", ""
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
      "sql.gcp.upbound.io/v1beta2/databaseinstances",
      "compute.gcp.upbound.io/v1beta2/instances",
      "container.gcp.upbound.io/v1beta2/clusters",
      "container.gcp.upbound.io/v1beta2/nodepools",
      "storage.gcp.upbound.io/v1beta2/buckets"
    ]
  }' \
  --cron "* * * * *"
```

**Overlap policy `Skip`:** if a scan takes longer than 1 minute (unlikely), the next
trigger is skipped rather than stacking concurrent scans.

---

## Advantages Over A/C

| Dimension | A/C (external binary) | D (Temporal Schedule) |
|---|---|---|
| New K8s Deployments | 1 (drift-watcher) | 0 |
| Scan observability | Pod logs | Temporal UI — full history per scan run with structured output |
| Alerting on scan failure | Log-based alerting | Temporal workflow failure alerting (already in place) |
| Update resource list | ConfigMap update + pod restart (A/C) | `temporal schedule update` — no deploy |
| Audit trail | Logs only | Full Temporal workflow history |

---

## Failure Modes

| Failure | Effect | Recovery |
|---|---|---|
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
- **Easiest resource type extension** — adding a new MR GVR only requires `temporal schedule update`; no deployment or restart
- **Reuses existing alerting** — scan failures surface as Temporal workflow failures, picked up by any existing Temporal alerting

### Cons
- **Fully coupled to Temporal availability** — if Temporal is down, drift scanning stops entirely; in A/C the watcher runs independently while only the workflow start call fails
- **Up to 1 min scan latency** — same as A's poll interval, worse than B/C event-driven approaches
- **Unusual pattern** — `ScanDriftActivity` calls back into Temporal to start `DriftApprovalWorkflow`; requires injecting a Temporal client into the activity
- **Scan history accumulates** — 1 `DriftScanWorkflow` entry per minute; requires a Temporal workflow retention policy to avoid unbounded storage growth
- **Potential rate limiting at scale** — if many resources drift simultaneously, all `DriftApprovalWorkflow` starts happen within one activity execution; may hit Temporal API rate limits with very large resource counts
