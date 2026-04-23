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
- The workflow is keyed on **MR name** — each drifted MR gets its own independent approval
workflow. A multi-MR XR (e.g. GKE = Cluster + NodePool) produces two separate workflows
if both MRs drift simultaneously.

For multi-MR XRs, the `AlreadyStarted` check deduplicates at the MR level — if the same
MR is still drifted on the next scan cycle, no duplicate workflow is started.

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

A resource is considered **drifted** when it satisfies **either** of two signals:

**Signal 1 — Field diff** (forProvider vs atProvider)
1. MR has label `platform.io/drift-protection: "true"`
2. MR `status.atProvider` is non-empty (Observe() has run at least once)
3. At least one field in `spec.forProvider` does not match the corresponding field
   in `status.atProvider`

**Signal 2 — Reconcile error** (fast path)
1. MR condition `Synced=False` with `reason=ReconcileError`

Signal 2 is checked first (fast path) because it catches resource deletion — when a
resource is deleted from GCP, `atProvider` retains stale values (Signal 1 would report
no diff), but the provider sets `Synced=False/ReconcileError` immediately.

### Comparison algorithm (Signal 1)

```
for each key in spec.forProvider:
    if key is in skipped list: continue
    if status.atProvider[key] != spec.forProvider[key]:
        record as drifted field

fields present in atProvider but absent in forProvider are ignored
(these are computed/read-only fields: id, selfLink, createTime, etc.)
```

This comparison is done recursively for nested objects and slices. The result is a list of
drifted field paths (e.g. `settings.tier`, `settings.diskSize`) passed to the
workflow as `DriftDetail`.

### Skipped forProvider keys

Some top-level `forProvider` keys are permanently excluded because GCP normalizes them
at read time, causing persistent false-positive diffs:

| Key | Reason skipped |
|-----|----------------|
| `bootDisk` | GCP resolves image family to a versioned URL in `atProvider` |
| `clusterRef` | Crossplane-only reference selector, never reflected in `atProvider` |
| `clusterSelector` | Crossplane-only reference selector, never reflected in `atProvider` |

### GCP URL normalization

GCP often returns full resource URLs in `atProvider` where `forProvider` uses short names
(e.g. `forProvider.network = "default"` vs `atProvider.network = ".../networks/default"`).
The comparison handles this: a value matches if `actual` has `"/"+desired` as a suffix.

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
// MR fields drive FlipManagementPolicyActivity and WaitMRReadyActivity.
// XR fields are used for the workflow ID, display, and logging.
// The watcher resolves the MR's ownerReferences to populate XR fields.
type DriftApprovalInput struct {
    // MR identification — used by FlipManagementPolicyActivity and WaitMRReadyActivity
    MRGroup     string `json:"mrGroup"`     // e.g. "sql.gcp.upbound.io"
    MRVersion   string `json:"mrVersion"`   // e.g. "v1beta2"
    MRResource  string `json:"mrResource"`  // e.g. "databaseinstances"
    MRName      string `json:"mrName"`      // e.g. "my-postgres-db-instance"
    MRNamespace string `json:"mrNamespace"` // empty for cluster-scoped MRs (current)

    // XR identification — used for workflow ID, display, and logging
    XRKind      string `json:"xrKind"`      // e.g. "XDatabase" (for display/logging)
    XRName      string `json:"xrName"`
    XRNamespace string `json:"xrNamespace"` // empty for cluster-scoped XRs (current)

    DetectedAt  string `json:"detectedAt"`  // RFC3339
    DriftDetail string `json:"driftDetail"` // human-readable diff, e.g. "settings.tier: want="db-n1-standard-2" got="db-n1-standard-4""
}
```

### FlipManagementPolicyInput

```go
// FlipManagementPolicyInput targets the MR directly.
// Pass Policies: ["Create","Observe","Update"] to unlock; ["Observe"] to re-lock.
type FlipManagementPolicyInput struct {
    MRGroup     string   `json:"mrGroup"`
    MRVersion   string   `json:"mrVersion"`
    MRResource  string   `json:"mrResource"`
    MRName      string   `json:"mrName"`
    MRNamespace string   `json:"mrNamespace"` // empty for cluster-scoped MRs (current)
    Policies    []string `json:"policies"`
}
```

### WaitMRReadyInput

```go
// WaitMRReadyInput waits at MR level using the same drift detection logic as
// ScanDriftActivity. Recovery is confirmed when isDrifted() returns false —
// meaning Crossplane successfully reconciled the MR back to desired state.
// Works for any MR type — no deletion side effects.
type WaitMRReadyInput struct {
    MRGroup     string        `json:"mrGroup"`
    MRVersion   string        `json:"mrVersion"`
    MRResource  string        `json:"mrResource"`
    MRName      string        `json:"mrName"`
    MRNamespace string        `json:"mrNamespace"` // empty for cluster-scoped MRs (current)
    Timeout     time.Duration `json:"timeout"`
}
```

---

## 7. DriftApprovalWorkflow (shared — all approaches fire this)

**File:** `backend/temporal-worker/internal/workflows/drift_approval.go`

**Workflow ID convention:**
- Cluster-scoped XR (current): `drift-approval-<xrkind>-<xrname>-<mrname>`
- Namespace-scoped XR (future): `drift-approval-<xrnamespace>-<xrkind>-<xrname>-<mrname>`

Keyed on **MR name** — one approval workflow per drifted MR. A multi-MR XR (e.g. GKE
Cluster + NodePool) produces two independent workflows if both MRs drift. Dedup is
enforced by Temporal: if the same MR is still drifted on the next scan cycle, the
`AlreadyStarted` error is silently swallowed and no duplicate is started.

The MR is already in `managementPolicies: ["Observe"]` when the workflow starts —
no flip is needed on entry.

```
STATE MACHINE
═════════════

NOTIFYING
  Action: NotifyDriftActivity (log stub — swap for Slack/PD in production)
  Note: failure here is non-blocking, workflow continues

WAITING_FOR_APPROVAL
  Query: "approval-status" → "waiting_for_approval"
  Signal: "approval-signal" (existing ApprovalSignal type, unchanged)
  Timeout: 24h
    │
    ├─ Approved ──────────────────► RECONCILING
    │                                 R1: FlipManagementPolicyActivity(Policies=["Create","Observe","Update"])
    │                                     [unlock MR — Crossplane resumes reconciliation]
    │                                 R2: WaitMRReadyActivity(MR, timeout=30min)
    │                                     [polls isDrifted() every 10s — no drift = success]
    │                                     [capture error, do not return yet]
    │                                 R3: FlipManagementPolicyActivity(Policies=["Observe"])
    │                                     [ALWAYS runs — restores Observe mode]
    │                                 return R2 error if any → RECONCILE_FAILED
    │
    ├─ Rejected ──────────────────► DRIFT_IGNORED
    │                                 NonRetryableApplicationError("APPROVAL_REJECTED")
    │                                 (MR stays in Observe — drift is accepted/known)
    │
    └─ 24h timeout ───────────────► NonRetryableApplicationError("APPROVAL_TIMEOUT")
                                      → DRIFT_TIMEOUT
                                      (MR stays in Observe — watcher re-detects on next cycle)

SAFETY RULE: R3 executes even if R2 times out or errors. MR is never left in
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

Log stub called after drift is detected. Swap the body for Slack/PagerDuty in production —
the function signature and `NotifyDriftInput` type stay unchanged.

```go
type NotifyDriftInput struct {
    MRName      string `json:"mrName"`
    MRNamespace string `json:"mrNamespace"`
    XRKind      string `json:"xrKind"`
    XRName      string `json:"xrName"`
    XRNamespace string `json:"xrNamespace"`
    DriftDetail string `json:"driftDetail"` // e.g. `settings.tier: want="db-n1-standard-2" got="db-n1-standard-4"`
    DetectedAt  string `json:"detectedAt"`  // RFC3339
}

func NotifyDriftActivity(ctx context.Context, in NotifyDriftInput) error {
    // structured JSON log — single swap point for real notifications
    fmt.Printf(`{"level":"warn","event":"drift_detected","mr":"%s","xrKind":"%s","xrName":"%s","detail":"%s","detectedAt":"%s"}`+"\n",
        in.MRName, in.XRKind, in.XRName, in.DriftDetail, in.DetectedAt)
    return nil
}
```

### FlipManagementPolicyActivity

Patches the MR's `spec.managementPolicies` directly via the K8s dynamic client.
Does not touch the XR or XRD parameter. Namespace-aware: uses cluster-scoped client
when `MRNamespace` is empty (current Crossplane MRs), namespace-scoped when set.

```go
// Works for any MR type — DatabaseInstance, Instance, Cluster, NodePool, Bucket, OmniaDatabase.
func FlipManagementPolicyActivity(ctx context.Context, in FlipManagementPolicyInput) error {
    dc, _ := k8s.NewDynamicClient()
    gvr := schema.GroupVersionResource{Group: in.MRGroup, Version: in.MRVersion, Resource: in.MRResource}
    patch := map[string]interface{}{"spec": map[string]interface{}{"managementPolicies": in.Policies}}
    patchBytes, _ := json.Marshal(patch)
    ri := dc.Resource(gvr)
    if in.MRNamespace != "" {
        _, err = ri.Namespace(in.MRNamespace).Patch(ctx, in.MRName, types.MergePatchType, patchBytes, metav1.PatchOptions{})
    } else {
        _, err = ri.Patch(ctx, in.MRName, types.MergePatchType, patchBytes, metav1.PatchOptions{})
    }
    return err
}
```

Usage in `DriftApprovalWorkflow`:
- Unlock: `Policies: []string{"Create", "Observe", "Update"}`
- Re-lock: `Policies: []string{"Observe"}`

### WaitMRReadyActivity

Polls the MR directly using the same `isDrifted()` logic as `ScanDriftActivity`.
Recovery is confirmed when no drift is detected — no deletion side effects.

```go
// Works for any MR type — same GVR coordinates used by FlipManagementPolicyActivity.
func WaitMRReadyActivity(ctx context.Context, in WaitMRReadyInput) error {
    dc, _ := k8s.NewDynamicClient()
    ri := dc.Resource(schema.GroupVersionResource{Group: in.MRGroup, Version: in.MRVersion, Resource: in.MRResource})
    deadline := time.Now().Add(in.Timeout)
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if time.Now().After(deadline) {
                return fmt.Errorf("timeout waiting for MR %s to become drift-free", in.MRName)
            }
            var obj *unstructured.Unstructured
            if in.MRNamespace != "" {
                obj, _ = ri.Namespace(in.MRNamespace).Get(ctx, in.MRName, metav1.GetOptions{})
            } else {
                obj, _ = ri.Get(ctx, in.MRName, metav1.GetOptions{})
            }
            drifted, _ := isDrifted(obj)
            if !drifted {
                return nil // Crossplane reconciled successfully
            }
        }
    }
}
```

---

## 9. WaitMRReadyActivity Timeout by Resource Type

`WaitMRReadyActivity` polls until `isDrifted()` returns false (drift resolved). Timeouts
are based on how long Crossplane typically takes to reconcile a resource after management
policies are restored. Current implementation uses **30 min** for all types (safe default).

| Resource | Typical reconcile time after policy restore | Recommended timeout |
|---|---|---|
| CloudSQL DatabaseInstance | 10–20 min (recreation) | 35 min |
| GCE Instance | 2–5 min | 15 min |
| GKE Cluster | 10–20 min | 35 min |
| GKE NodePool | 5–10 min | 20 min |
| GCS Bucket | <1 min | 5 min |

If `WaitMRReadyActivity` times out, R3 (`FlipManagementPolicyActivity` back to `["Observe"]`)
still runs — the MR is never left in full management mode.

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
