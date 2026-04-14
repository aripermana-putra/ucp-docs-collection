---
title: "Shared Design"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Shared Design — All Approaches

All four drift detection approaches share the same Crossplane changes, Temporal workflow,
and activity contracts. Only the trigger mechanism differs.

---

## 1. Core Concept: Watch at MR Level, Resolve to XR

Drift is detectable by comparing what Crossplane declared (`spec.forProvider`) against
what it actually observed from the cloud provider (`status.atProvider`). These fields
live on the **Managed Resource (MR)**, not on the Composite Resource (XR).

```
External resource modified (e.g. CloudSQL tier changed in GCP Console)
    │
    └── Crossplane provider calls Observe() on next poll
            └── MR.status.atProvider.settings.tier = "db-n1-standard-4"  ← actual
                MR.spec.forProvider.settings.tier  = "db-n1-standard-2"  ← desired
                                                     ↑ DIFF DETECTED HERE
                    └── resolve MR.metadata.ownerReferences → XR name
                            └── fire DriftApprovalWorkflow(xrName, mrName, diff)
```

Watching at MR level and resolving to XR via `ownerReferences` is the correct approach
because:
- `spec.forProvider` and `status.atProvider` only exist on MRs, not XRs
- XRs only carry aggregated conditions — the actual field values are on MRs
- The workflow is keyed on XR name (one approval per resource, not per MR)

For multi-MR XRs (e.g. GKE = Cluster + NodePool), both MRs resolve to the same XR.
The `AlreadyStarted` check in the watcher deduplicates — only one workflow runs per XR.

| Resource type | XR kind | MR kind(s) watched |
|---|---|---|
| CloudSQL | XDatabase | DatabaseInstance |
| GCE Compute | XComputeInstance | Instance |
| GKE | XKubernetesCluster | Cluster, NodePool |
| GCS | XObjectStorage | Bucket |
| Omnia (private) | XDatabase | OmniaDatabase *(see section 10)* |

### Why `status.atProvider` is always populated

Even under `managementPolicies: ["Observe"]`, Crossplane still calls `Observe()` on every
poll cycle. The provider reads the actual external state and writes it to `status.atProvider`.
Only `Create`, `Update`, and `Delete` are skipped. This means the drift data is always
fresh and available for comparison.

**Source:** `crossplane-runtime v1.17.0 / v2.2.0`
`pkg/reconciler/managed/reconciler.go` — `Observe()` is called unconditionally before
the management policy check.

---

## 2. Drift Protection Opt-in

Two mechanisms work together:

### 2a. Label on the XR — consumer opt-in signal

```yaml
metadata:
  labels:
    platform.io/drift-protection: "true"
```

Set by the consumer on the XR. The composition propagates this label to all composed MRs
(see section 4). Watchers use the MR label to filter which resources to monitor.

`platform.io/drift-protection` is a custom label — it has no built-in Crossplane meaning.
It exists to distinguish drift-protected resources from those using `managementPolicies:
["Observe"]` for other reasons (e.g. importing existing resources).

### 2b. XRD parameter — enables Observe mode on MRs

```yaml
# In XR spec.parameters:
driftProtection: true
```

When `true`, the composition renders `managementPolicies: ["Observe"]` on composed MRs,
blocking Crossplane from auto-healing while keeping `Observe()` active for drift detection.

**Separation of concerns:**
- Label = watcher filter: "watch this resource for drift"
- XRD param = reconciliation control: "block Crossplane from auto-healing this resource"

Both are set together when enabling drift protection. The composition handles propagating
both to the MR level.

---

## 3. XRD Changes (all resource types)

Add to `spec.parameters.properties` in each XRD:

```yaml
driftProtection:
  type: boolean
  description: |
    When true, composed managed resources use managementPolicies: [Observe].
    Crossplane still observes the resource (atProvider is kept up to date) but
    does not auto-heal. Drift is detected by comparing forProvider vs atProvider
    and triggers a Temporal human approval workflow.
    Set alongside label platform.io/drift-protection=true on the XR.
  default: false
```

Files to modify:
- `crossplane/xrd/dbaas/xdatabase.xrd.yaml`
- `crossplane/xrd/compute/xcomputeinstance.xrd.yaml`
- `crossplane/xrd/kubernetes/xkubernetescluster.xrd.yaml`
- `crossplane/xrd/objectstorage/xobjectstorage.xrd.yaml`

---

## 4. Composition Changes

In each composition's go-template, propagate both the label and management policy
to the primary managed resource:

```yaml
metadata:
  name: {{ $name }}
  labels:
    {{- if $p.driftProtection }}
    # Propagated from XR — watcher uses this label to filter MRs to monitor
    platform.io/drift-protection: "true"
    {{- end }}
spec:
  {{- if $p.driftProtection }}
  # Observe() still runs (atProvider updated); Create/Update/Delete are blocked
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

1. MR has label `platform.io/drift-protection: "true"`
2. MR `status.atProvider` is non-empty (Observe() has run at least once)
3. At least one field in `spec.forProvider` does not match the corresponding field
   in `status.atProvider`

### Comparison algorithm

```
for each key in spec.forProvider:
    if status.atProvider[key] != spec.forProvider[key]:
        record as drifted field

fields present in atProvider but absent in forProvider are ignored
(these are computed/read-only fields: id, selfLink, createTime, etc.)
```

This comparison is done recursively for nested objects. The result is a list of
drifted field paths (e.g. `settings.tier`, `settings.diskSize`) passed to the
workflow as `DriftDetail`.

### Confirmed atProvider coverage per GCP resource type

Verified against official provider CRD schemas in `upbound/provider-gcp`:

| MR kind | Key driftable fields in `atProvider` |
|---|---|
| `DatabaseInstance` | `databaseVersion`, `settings.tier`, `settings.diskSize`, `settings.ipConfiguration` |
| `Instance` (Compute) | `machineType`, `zone`, `disks`, `metadata`, `tags` |
| `Cluster` (GKE) | `addonsConfig`, `initialNodeCount`, `location`, `nodeConfig` |
| `NodePool` (GKE) | `initialNodeCount`, `nodeConfig`, `autoscaling` |
| `Bucket` (GCS) | `storageClass`, `location`, `versioning`, `lifecycleRule`, `labels` |

---

## 6. Temporal Types (shared across all approaches)

### DriftApprovalInput

```go
// DriftApprovalInput carries both MR and XR identification.
// MR fields are needed to patch managementPolicies (FlipManagementPolicyActivity).
// XR fields are needed for workflow ID, display, and WaitXRReadyActivity.
// The watcher resolves the MR's ownerReferences to populate XR fields.
type DriftApprovalInput struct {
    // MR identification — used by FlipManagementPolicyActivity
    MRGroup     string `json:"mrGroup"`     // e.g. "sql.gcp.upbound.io"
    MRVersion   string `json:"mrVersion"`   // e.g. "v1beta2"
    MRResource  string `json:"mrResource"`  // e.g. "databaseinstances"
    MRName      string `json:"mrName"`      // e.g. "my-postgres-db-instance"
    MRNamespace string `json:"mrNamespace"`

    // XR identification — used for workflow ID, display, and WaitXRReadyActivity
    XRGroup     string `json:"xrGroup"`    // e.g. "platform.example.io"
    XRVersion   string `json:"xrVersion"`  // e.g. "v1alpha1"
    XRResource  string `json:"xrResource"` // e.g. "xdatabases"
    XRKind      string `json:"xrKind"`     // e.g. "XDatabase" (for display/logging)
    XRName      string `json:"xrName"`
    XRNamespace string `json:"xrNamespace"`

    DetectedAt  string `json:"detectedAt"`  // RFC3339
    DriftDetail string `json:"driftDetail"` // human-readable diff, e.g. "settings.tier: db-n1-standard-2 → db-n1-standard-4"
}
```

### FlipManagementPolicyInput

```go
// FlipManagementPolicyInput targets the MR directly.
// Enable=true  → managementPolicies: ["Observe"]  (block auto-heal, keep observing)
// Enable=false → managementPolicies: ["*"]         (full management, allow auto-heal)
type FlipManagementPolicyInput struct {
    MRGroup     string `json:"mrGroup"`
    MRVersion   string `json:"mrVersion"`
    MRResource  string `json:"mrResource"`
    MRName      string `json:"mrName"`
    MRNamespace string `json:"mrNamespace"`
    Enable      bool   `json:"enable"`
}
```

### WaitXRReadyInput

```go
// WaitXRReadyInput waits at XR level — XR conditions (Ready, Synced) are valid
// once managementPolicies is restored to full management and Crossplane reconciles.
// Works for any XR type — all share the same Crossplane condition schema.
type WaitXRReadyInput struct {
    XRGroup     string        `json:"xrGroup"`
    XRVersion   string        `json:"xrVersion"`
    XRResource  string        `json:"xrResource"`
    XRName      string        `json:"xrName"`
    XRNamespace string        `json:"xrNamespace"`
    Timeout     time.Duration `json:"timeout"`
}
```

---

## 7. DriftApprovalWorkflow (shared — all approaches fire this)

**File:** `backend/temporal-worker/internal/workflows/drift_approval.go`

**Workflow ID convention:** `drift-approval-<xrNamespace>-<xrKind>-<xrName>`

Keyed on XR, not MR — one approval workflow per resource regardless of how many MRs
drifted underneath it. AlreadyStarted errors from multi-MR XRs are silently ignored
by all watchers.

The MR is already in `managementPolicies: ["Observe"]` when the workflow starts —
no flip is needed on entry.

```
STATE MACHINE
═════════════

NOTIFYING
  Action: NotifyDriftActivity(event=DRIFT_DETECTED)
  Note: failure here is non-blocking, workflow continues

WAITING_FOR_APPROVAL
  Query: "approval-status" → "waiting_for_approval"
  Signal: "approval-signal" (existing ApprovalSignal type, unchanged)
  Timeout: 24h
    │
    ├─ Approved ──────────────────► RECONCILING
    │                                 R1: FlipManagementPolicyActivity(MR, Enable=false)
    │                                     [lift Observe → full management]
    │                                     [Crossplane resumes reconciliation, fixes drift]
    │                                 R2: WaitXRReadyActivity(XR, timeout=configurable)
    │                                     [capture error, do not return yet]
    │                                 R3: FlipManagementPolicyActivity(MR, Enable=true)
    │                                     [ALWAYS runs — restores Observe mode]
    │                                 if R2 error:
    │                                   R4: NotifyDriftActivity(event=RECONCILIATION_FAILED)
    │                                       ["Approved reconciliation failed — manual check required"]
    │                                   return R2 error → RECONCILE_FAILED
    │
    ├─ Rejected ──────────────────► DRIFT_IGNORED
    │                                 NonRetryableApplicationError("APPROVAL_REJECTED")
    │                                 (MR stays in Observe — drift is accepted/known)
    │
    └─ 24h timeout ───────────────► NotifyDriftActivity(event=APPROVAL_TIMEOUT)
                                      ["No approval in 24h — drift still unaddressed"]
                                      NonRetryableApplicationError("APPROVAL_TIMEOUT")
                                      → DRIFT_TIMEOUT
                                      (MR stays in Observe — watcher will re-detect)

SAFETY RULE: R3 executes even if R2 times out or errors. MR is never left in
             full management mode (unprotected) longer than one activity window.

NOTIFICATION EVENTS
  DRIFT_DETECTED        — initial alert when forProvider vs atProvider diff is found
  APPROVAL_TIMEOUT      — nobody approved within 24h; watcher re-detects on next cycle
  RECONCILIATION_FAILED — approved reconciliation timed out or failed;
                          MR is back in Observe mode but still drifted
```

**Reused without modification from existing codebase:**
- `waitForApproval()` from `request_database.go`
- `"approval-signal"` signal name and `ApprovalSignal` type
- `"approval-status"` query handler pattern

---

## 8. Activities (shared — all approaches use these)

**File:** `backend/temporal-worker/internal/activities/drift.go`

### NotifyDriftActivity

Called at three points in the workflow: initial detection, approval timeout, and
reconciliation failure. The `Event` field distinguishes them.

```go
// NotifyDriftInput carries context for all three notification events.
type NotifyDriftInput struct {
    XRKind      string `json:"xrKind"`
    XRName      string `json:"xrName"`
    Namespace   string `json:"namespace"`
    DetectedAt  string `json:"detectedAt"`   // RFC3339
    DriftDetail string `json:"driftDetail"`  // e.g. "settings.tier: db-n1-standard-2 → db-n1-standard-4"
    // Event distinguishes why the notification is being sent.
    // Values: "DRIFT_DETECTED" | "APPROVAL_TIMEOUT" | "RECONCILIATION_FAILED"
    Event       string `json:"event"`
}

// POC stub. Swap body for Slack + email + PagerDuty in production.
// Function signature and NotifyDriftInput type remain unchanged.
func NotifyDriftActivity(ctx context.Context, in NotifyDriftInput) error {
    activity.GetLogger(ctx).Info("drift notification",
        "event", in.Event,
        "xrKind", in.XRKind,
        "xrName", in.XRName,
        "namespace", in.Namespace,
        "detectedAt", in.DetectedAt,
        "driftDetail", in.DriftDetail,
    )
    return nil
}
```

### FlipManagementPolicyActivity

Patches the MR's `spec.managementPolicies` directly via the K8s dynamic client.
Does not touch the XR or XRD parameter.

```go
// Works for any MR type — DatabaseInstance, Instance, Cluster, NodePool, Bucket, OmniaDatabase.
func FlipManagementPolicyActivity(ctx context.Context, in FlipManagementPolicyInput) error {
    dc, _ := k8s.NewDynamicClient()
    gvr := schema.GroupVersionResource{
        Group:    in.MRGroup,
        Version:  in.MRVersion,
        Resource: in.MRResource,
    }
    policies := []string{"*"}
    if in.Enable {
        policies = []string{"Observe"}
    }
    patch := map[string]interface{}{
        "spec": map[string]interface{}{
            "managementPolicies": policies,
        },
    }
    patchBytes, _ := json.Marshal(patch)
    _, err := dc.Resource(gvr).Namespace(in.MRNamespace).Patch(
        ctx, in.MRName, types.MergePatchType, patchBytes, metav1.PatchOptions{},
    )
    return err
}
```

### WaitXRReadyActivity

Waits at XR level — XR conditions (Ready, Synced) are aggregated from all composed MRs
by the Crossplane composite reconciler and are valid once full management is restored.

```go
// Generic: works for XDatabase, XComputeInstance, XKubernetesCluster, XObjectStorage.
func WaitXRReadyActivity(ctx context.Context, in WaitXRReadyInput) error {
    dc, _ := k8s.NewDynamicClient()
    gvr := schema.GroupVersionResource{
        Group:    in.XRGroup,
        Version:  in.XRVersion,
        Resource: in.XRResource,
    }
    deadline := time.Now().Add(in.Timeout)
    for time.Now().Before(deadline) {
        obj, _ := dc.Resource(gvr).Namespace(in.XRNamespace).Get(ctx, in.XRName, metav1.GetOptions{})
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
|---|---|---|
| CloudSQL | 10–20 min | 35 min |
| GCE Compute | 2–5 min | 15 min |
| GKE Cluster | 10–20 min | 35 min |
| GCS Bucket | <1 min | 5 min |

---

## 10. Omnia (provider-roc) Pending Requirement

The `forProvider` vs `atProvider` diff approach requires that `status.atProvider`
mirrors config fields from `spec.forProvider`. For GCP resources this is already
the case (confirmed from `upbound/provider-gcp` CRD schemas).

For `OmniaDatabase`, the current `OmniaDatabaseObservation` struct only contains
connection info (endpoint, port, deploymentId, status) — it does not mirror config
fields. This makes `forProvider` vs `atProvider` diff impossible as-is.

**Required change to `provider-roc`:**

```go
// Current — only connection/status info
type OmniaDatabaseObservation struct {
    DeploymentID string `json:"deploymentId,omitempty"`
    Endpoint     string `json:"endpoint,omitempty"`
    Port         int    `json:"port,omitempty"`
    Status       string `json:"status,omitempty"`
    State        string `json:"state,omitempty"`
    Error        string `json:"error,omitempty"`
}

// Required — add mirrors of driftable forProvider fields
type OmniaDatabaseObservation struct {
    // ... existing fields unchanged ...

    // Observed config — populated by Observe() from Omnia API response.
    // Used by the drift detection watcher to diff against spec.forProvider.
    Middleware        *string `json:"middleware,omitempty"`
    MiddlewareVersion *string `json:"middlewareVersion,omitempty"`
    Topology          *ObservedTopology       `json:"topology,omitempty"`
    Configuration     *ObservedConfiguration  `json:"configuration,omitempty"`
    BackupTime        *string `json:"backupTime,omitempty"`
}
```

The `Observe()` function in `provider-roc` must populate these fields from the
Omnia API response, and `IsUpToDate()` must compare them against `forProvider`.

**This is not blocking the POC.** The drift detection design is unified from day one.
Omnia support is a tracked implementation task for the provider-roc.

---

## 11. Example: Enabling Drift Protection on a Resource

```yaml
apiVersion: platform.example.io/v1alpha1
kind: XDatabase
metadata:
  name: my-postgres
  namespace: default
  labels:
    platform.io/drift-protection: "true"   # opt-in to drift monitoring
spec:
  parameters:
    driftProtection: true                  # composition sets managementPolicies: [Observe] on MRs
    engine: postgres
    engineVersion: "15"
```

```bash
# Verify composed DatabaseInstance is in Observe mode:
kubectl get databaseinstance -A -o jsonpath='{.items[0].spec.managementPolicies}'
# Expected: ["Observe"]

# Verify atProvider is populated (Observe() has run):
kubectl get databaseinstance -A -o jsonpath='{.items[0].status.atProvider}'

# Simulate drift: change tier in GCP Console, then wait for next poll (~10 min).
# Watcher will detect: spec.forProvider.settings.tier != status.atProvider.settings.tier
```
