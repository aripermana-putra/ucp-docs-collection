---
title: "Shared Design"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Shared Design — All Approaches

All four drift detection approaches share the same Crossplane changes, Temporal workflow,
and activity contracts. Only the trigger mechanism differs.

---

## 1. Core Concept: Watch at XR Level

Drift propagates upward through Crossplane's resource hierarchy:

```
GCP resource deleted/modified (e.g. CloudSQL instance)
    └── DatabaseInstance.status.conditions[Synced] = False
            └── XDatabase.status.conditions[Synced] = False   ← WATCH HERE
```

By watching **Composite Resources (XRs)** instead of individual Managed Resources (MRs),
the same logic handles every resource type automatically:

| Resource type | XR kind | MR kind(s) |
|---------------|---------|------------|
| CloudSQL | XDatabase | DatabaseInstance, Database, User |
| GCE Compute | XComputeInstance | Instance |
| GKE | XKubernetesCluster | Cluster, NodePool |
| GCS | XObjectStorage | Bucket |
| Future (AWS RDS, Azure, LB, VPC…) | new XRD | new MRs |

Adding a new cloud product = new XRD + new composition. Drift detection code unchanged.

---

## 2. Drift Protection Opt-in

Two mechanisms are used together:

### 2a. Label on the XR — for targeting

```yaml
metadata:
  labels:
    platform.io/drift-protection: "true"
```

Applied to any XR to opt into drift monitoring. No XRD changes required for this.
Watchers/scanners use this label to filter which resources to monitor.

### 2b. XRD parameter — for ManagementPolicy control

```yaml
# In XR spec.parameters:
driftProtection: true
```

When `true`, the composition's go-template renders `managementPolicies: ["Observe"]` on
the composed managed resources, preventing Crossplane from auto-healing.

**Separation of concerns:**
- Label = "is this resource being monitored for drift?"
- XRD param = "is Crossplane blocked from reconciling this resource?"

Normally both are set together when enabling drift protection on a resource.

---

## 3. XRD Changes (all resource types)

Add to `spec.parameters.properties` in each XRD:

```yaml
driftProtection:
  type: boolean
  description: |
    When true, composed managed resources use managementPolicies: [Observe].
    Crossplane detects drift but does not auto-heal. Drift triggers a Temporal
    approval workflow. Set alongside label platform.io/drift-protection=true.
  default: false
```

Files to modify:
- `crossplane/xrd/dbaas/xdatabase.xrd.yaml`
- `crossplane/xrd/compute/xcomputeinstance.xrd.yaml`
- `crossplane/xrd/kubernetes/xkubernetescluster.xrd.yaml`
- `crossplane/xrd/objectstorage/xobjectstorage.xrd.yaml`

---

## 4. Composition Changes

In each composition's go-template, add to the primary managed resource:

```yaml
metadata:
  name: {{ $name }}
  labels:
    {{- if $p.driftProtection }}
    platform.io/drift-protection: "true"
    {{- end }}
spec:
  {{- if $p.driftProtection }}
  managementPolicies:
  - Observe
  {{- end }}
  forProvider:
    # ... existing fields unchanged
```

Files to modify:
- `crossplane/composition/dbaas/cloudsql/xdatabase-cloudsql.yaml` → DatabaseInstance
- `crossplane/composition/compute/gcp/xcomputeinstance-gce.yaml` → Instance
- `crossplane/composition/kubernetes/gke/xkubernetescluster-gke.yaml` → Cluster + NodePool
- `crossplane/composition/objectstorage/gcs/xobjectstorage-gcs.yaml` → Bucket

---

## 5. Drift Detection Criteria

A resource is considered **drifted** when ALL of:

1. XR has label `platform.io/drift-protection: "true"`
2. XR `status.conditions[Synced].status == "False"`
3. XR `status.conditions[Synced].reason` is `ReconcileError`

The only official Synced reasons in crossplane-runtime are `ReconcileSuccess`, `ReconcileError`,
and `ReconcilePaused` (pause annotation only — not related to `managementPolicies: ["Observe"]`).
Condition 1 gates on provisioned, drift-protected resources only.

---

## 6. Temporal Types (shared across all approaches)

### DriftApprovalInput

```go
// DriftApprovalInput identifies the drifted XR and carries context for the workflow.
// XRGroup/Version/Resource identify the Kubernetes API so the workflow can operate on
// any XR type without type-specific branching.
type DriftApprovalInput struct {
    XRGroup     string `json:"xrGroup"`    // e.g. "platform.example.io"
    XRVersion   string `json:"xrVersion"`  // e.g. "v1alpha1"
    XRResource  string `json:"xrResource"` // e.g. "xdatabases", "xcomputeinstances"
    XRKind      string `json:"xrKind"`     // e.g. "XDatabase" (for display/logging)
    XRName      string `json:"xrName"`
    Namespace   string `json:"namespace"`
    DetectedAt  string `json:"detectedAt"`  // RFC3339
    DriftReason string `json:"driftReason"` // Synced condition reason
}
```

### FlipManagementPolicyInput

```go
// Enable=true → driftProtection=true → composition renders managementPolicies=[Observe]
// Enable=false → driftProtection=false → composition renders full management (*)
type FlipManagementPolicyInput struct {
    XRGroup    string `json:"xrGroup"`
    XRVersion  string `json:"xrVersion"`
    XRResource string `json:"xrResource"`
    XRName     string `json:"xrName"`
    Namespace  string `json:"namespace"`
    Enable     bool   `json:"enable"` // true=Observe, false=full management
}
```

### WaitXRReadyInput

```go
// Generic replacement for per-type wait activities.
// Works for XDatabase, XComputeInstance, XKubernetesCluster, XObjectStorage, and any
// future XR type — all share the same Crossplane Ready/Synced condition schema.
type WaitXRReadyInput struct {
    XRGroup    string        `json:"xrGroup"`
    XRVersion  string        `json:"xrVersion"`
    XRResource string        `json:"xrResource"`
    XRName     string        `json:"xrName"`
    Namespace  string        `json:"namespace"`
    Timeout    time.Duration `json:"timeout"`
}
```

---

## 7. DriftApprovalWorkflow (shared — all approaches fire this)

**File:** `backend/temporal-worker/internal/workflows/drift_approval.go`

**Workflow ID convention:** `drift-approval-<namespace>-<xrKind>-<xrName>`

```
STATE MACHINE
═════════════

NOTIFYING
  Action: NotifyDriftActivity (structured log — POC stub)
  Note: failure here is non-blocking, workflow continues

WAITING_FOR_APPROVAL
  Query: "approval-status" → "waiting_for_approval"
  Signal: "approval-signal" (existing ApprovalSignal type, unchanged)
  Timeout: 24h
    │
    ├─ Approved ──────────────────► RECONCILING
    │                                 R1: FlipManagementPolicyActivity(Enable=false)
    │                                     [lifts Observe → full management]
    │                                 R2: WaitXRReadyActivity(timeout=configurable)
    │                                     [capture error, do not return yet]
    │                                 R3: FlipManagementPolicyActivity(Enable=true)
    │                                     [ALWAYS runs — restores Observe mode]
    │                                 Return R2 error if any
    │
    ├─ Rejected ──────────────────► DRIFT_IGNORED
    │                                 NonRetryableApplicationError("APPROVAL_REJECTED")
    │
    └─ 24h timeout ───────────────► DRIFT_TIMEOUT
                                      NonRetryableApplicationError("APPROVAL_TIMEOUT")

SAFETY RULE: R3 executes even if R2 times out. Resource is never left in
             full management mode (unprotected) longer than one activity window.
```

**Reused without modification from existing codebase:**
- `waitForApproval()` from `request_database.go`
- `"approval-signal"` signal name and `ApprovalSignal` type
- `"approval-status"` query handler pattern

---

## 8. Activities (shared — all approaches use these)

**File:** `backend/temporal-worker/internal/activities/drift.go`

### NotifyDriftActivity

```go
// POC stub. Swap body for Slack + email + PagerDuty in production.
// Function signature and NotifyDriftInput type remain unchanged.
func NotifyDriftActivity(ctx context.Context, in NotifyDriftInput) error {
    activity.GetLogger(ctx).Info("DRIFT DETECTED — human approval required",
        "xrKind", in.XRKind,
        "xrName", in.XRName,
        "namespace", in.Namespace,
        "detectedAt", in.DetectedAt,
        "driftReason", in.DriftReason,
    )
    return nil
}
```

### FlipManagementPolicyActivity

```go
// Generic: works for XDatabase, XComputeInstance, XKubernetesCluster, XObjectStorage.
func FlipManagementPolicyActivity(ctx context.Context, in FlipManagementPolicyInput) error {
    dc, _ := k8s.NewDynamicClient()
    gvr := schema.GroupVersionResource{
        Group:    in.XRGroup,
        Version:  in.XRVersion,
        Resource: in.XRResource,
    }
    patch := map[string]interface{}{
        "spec": map[string]interface{}{
            "parameters": map[string]interface{}{
                "driftProtection": in.Enable,
            },
        },
    }
    patchBytes, _ := json.Marshal(patch)
    _, err = dc.Resource(gvr).Namespace(in.Namespace).Patch(
        ctx, in.XRName, types.MergePatchType, patchBytes, metav1.PatchOptions{},
    )
    return err
}
```

### WaitXRReadyActivity

```go
// Generic replacement for WaitDatabaseReadyActivity, WaitComputeInstanceReadyActivity, etc.
func WaitXRReadyActivity(ctx context.Context, in WaitXRReadyInput) error {
    dc, _ := k8s.NewDynamicClient()
    gvr := schema.GroupVersionResource{
        Group:    in.XRGroup,
        Version:  in.XRVersion,
        Resource: in.XRResource,
    }
    deadline := time.Now().Add(in.Timeout)
    for time.Now().Before(deadline) {
        obj, _ := dc.Resource(gvr).Namespace(in.Namespace).Get(ctx, in.XRName, metav1.GetOptions{})
        // check obj.status.conditions[Ready].status == "True"
        // check for ReconcileError → return NonRetryableApplicationError
        time.Sleep(15 * time.Second)
    }
    return fmt.Errorf("timeout waiting for %s/%s to be ready", in.XRResource, in.XRName)
}
```

---

## 9. Recommended Wait Timeouts by Resource Type

| Resource | Typical provisioning time | Recommended timeout |
|----------|--------------------------|---------------------|
| CloudSQL | 10–20 min | 35 min |
| GCE Compute | 2–5 min | 15 min |
| GKE Cluster | 10–20 min | 35 min |
| GCS Bucket | <1 min | 5 min |

---

## 10. Example: Enabling Drift Protection on a Resource

```yaml
apiVersion: platform.example.io/v1alpha1
kind: XDatabase
metadata:
  name: my-postgres
  namespace: default
  labels:
    platform.io/drift-protection: "true"   # enables monitoring
spec:
  parameters:
    driftProtection: true                  # enables Observe mode on composed MRs
    engine: postgres
    engineVersion: "15"
```

```bash
# Verify composed DatabaseInstance is in Observe mode:
kubectl get databaseinstance -A -o jsonpath='{.items[0].spec.managementPolicies}'
# Expected: ["Observe"]
```
