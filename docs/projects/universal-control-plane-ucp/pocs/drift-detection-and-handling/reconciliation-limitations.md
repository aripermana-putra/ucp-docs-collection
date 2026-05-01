---
title: "Drift Reconciliation Limitations"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Drift Reconciliation Limitations

This is a living document. It captures every drift scenario where the platform either cannot
**detect** the drift or cannot **recover** from it. Each entry is added after the scenario has
been tested end-to-end against the running platform.

Two categories of limitation are tracked:

- **Detection Limitations** — the `DriftApprovalWorkflow` never fires; the platform's detection
  signals do not trigger even though real drift has occurred.
- **Confirmed Findings (Recovery Failures)** — drift is detected and the operator is notified,
  but after approval Crossplane attempts to restore the resource and the cloud provider permanently
  rejects the change.

**What the platform guarantees even when recovery fails:**
- `WaitMRReadyActivity` detects the persistent `ReconcileError` after a 2-minute grace
  period and surfaces the exact cloud error message in the Temporal UI
- `managementPolicies` is always flipped back to `[Observe]` — the MR is never left in
  full management mode regardless of recovery outcome

---

## Detection Limitations

These are scenarios where drift **cannot be detected** — the `DriftApprovalWorkflow` never
fires because the platform's detection signals do not trigger.

---

### DET-01 — Upjet Late Initialization Overwrites Boolean Zero-Value Desired State

| | |
|---|---|
| **Cloud Provider** | GCP (all upjet-based providers) |
| **Service** | Any |
| **Affected Fields** | Any `boolean` field in `spec.forProvider` where desired state is `false` |
| **Example** | GCS `uniformBucketLevelAccess: false` |
| **Detectable** | No |
| **Source** | MCUCP-181 (drift-reconcile-sim) |

**Scenario**

A GCS bucket has `uniformBucketLevelAccess: false` as the desired state in `spec.forProvider`.
Someone enables it externally (GCP Console or `gcloud`). Drift is expected to be detected via
Signal 1 (forProvider vs atProvider diff: `desired=false, actual=true`).

**What actually happens**

Crossplane's `Observe()` cycle runs late initialization on every reconcile loop. For any field in
`spec.forProvider` that equals the Go zero value (`false` for bool, `0` for int, `""` for string),
upjet treats it as "unset" and copies the observed `atProvider` value back into `forProvider`.

After the external mutation sets `uniformBucketLevelAccess: true`:

1. `atProvider.uniformBucketLevelAccess = true`
2. Late init fires: `forProvider.uniformBucketLevelAccess` was `false` (zero value) → overwritten to `true`
3. Diff: `forProvider=true, atProvider=true` → no drift detected

The correction closes in seconds — before the drift scan has a chance to see the mismatch.

**Why**

In Go, `false` is the zero value for `bool`. upjet cannot distinguish between "user explicitly
set this to false" and "user never set this field at all". The late-init heuristic assumes zero
value = unset and fills it in from the observed cloud state.

This affects **any boolean field where the desired state is `false`** across all upjet-based
Crossplane providers (provider-upjet-gcp, provider-upjet-aws, etc.).

**Impact on Signal 2**

Signal 2 (`Synced=False / ReconcileError`) does not fire in `Observe` mode — Crossplane does not
attempt writes, so there is no reconcile error to surface.

**Workaround**

None in Observe-mode drift protection. Fields with a `false` desired state are a blind spot: if
someone enables the feature externally, the platform sees no drift.

---

## Confirmed Findings

---

### LIM-01 — GCP / Cloud SQL: Disk Size Reduction

| | |
|---|---|
| **Cloud Provider** | GCP |
| **Service** | Cloud SQL |
| **MR Kind** | `DatabaseInstance` (`sql.gcp.upbound.io`) |
| **Drift Signal** | Signal 1 — field diff (`settings.disk_size`) |
| **Reconcilable** | No |
| **Source** | F-07, MCUCP-158 |

**Scenario**

The Cloud SQL instance disk size is manually increased in the GCP Console (e.g. 50 GB →
150 GB). After approval, Crossplane attempts to PATCH the instance back to 50 GB.

**Error**

```
update failed: async update failed: refuse to update the external resource because the
following update requires replacing it: cannot change the value of the argument
"settings.0.disk_size" from "150" to "50"
```

MR condition: `Synced=False, reason=ReconcileError, type=LastAsyncOperation: AsyncUpdateFailure`.
Persists indefinitely — `WaitMRReadyActivity` exits early with `RECONCILE_FAILED`.

**Why**

GCP Cloud SQL enforces that disk size can only grow, never shrink. This is a hard API
constraint — allocated storage cannot be reclaimed on a live instance.

**Operator Action**

- **Accept the new size (recommended):** Update `spec.forProvider.settings.disk_size` in the
  XDatabase claim to match the current GCP value. Crossplane will clear the `ReconcileError`.
  The larger disk is billed accordingly.
- **Recreate (data loss risk):** If the original size is a hard requirement, create a new
  instance at the desired size, migrate data, and decommission the old one. This requires
  planned downtime.
- **Revert in GCP:** Not possible — GCP does not allow disk size reduction.

---

### LIM-02 — GCP / GKE: Cluster or NodePool Deleted with Stale Late-Initialized Fields

| | |
|---|---|
| **Cloud Provider** | GCP |
| **Service** | GKE |
| **MR Kind** | `Cluster`, `NodePool` (`container.gcp.upbound.io`) |
| **Drift Signal** | Signal 2 — `Synced=False / ReconcileError` (resource not found) |
| **Reconcilable** | No (without manual preparation) |
| **Source** | F-01, F-07, MCUCP-158 |

**Scenario**

A GKE Cluster or NodePool is deleted from the GCP Console. After approval, Crossplane
attempts to recreate the resource using the current `spec.forProvider`. Recreation fails.

**Failure sequence observed (GKE Cluster):**

1. `Error 400: Cannot specify logging_config together with logging_service` — the
   composition had `loggingService: "none"` (deprecated); late initialization had also
   written `loggingConfig` (new field) from the original cluster's observed state into
   `forProvider`. GCP rejects both field generations in the same request.
2. After fixing the composition — `Error 400: SYSTEM_COMPONENTS monitoring must be
   enabled if any monitoring is enabled` — the late-initialized `monitoringConfig`
   was missing `SYSTEM_COMPONENTS`.
3. After fixing the composition again — Cluster creates. NodePool fails: pod secondary
   range `gke-ari-test-kube-cluster-1-pods-0f92ba7c` not found — the late-initialized
   `forProvider.networkConfig.podRange` referenced a range that belonged to the deleted
   cluster and no longer exists.

**Why**

After Crossplane creates a resource, the provider's `Observe()` cycle writes GCP's full
API response back into `spec.forProvider`, including fields the composition never
declared (pod secondary range names, auto-assigned component lists, etc.). These are
values specific to the original resource instance.

When the resource is deleted and Crossplane tries to recreate it, those fields reference
state that no longer exists. GCP rejects the request. This is a deletion-specific
failure — field-level drift on a live resource is not affected.

**Operator Action**

- **Delete the MR directly (recommended):** Deleting the composed MR forces Crossplane
  to recreate it via the composition with a clean `forProvider`, bypassing the stale
  late-initialized fields.
  ```bash
  kubectl delete cluster.container.gcp.upbound.io <mr-name>
  kubectl get cluster.container.gcp.upbound.io -w
  ```
- **Manually clean `forProvider`:** Edit the MR before approving to remove stale
  late-initialized fields. Error-prone and requires deep GKE API knowledge.

> **Longer-term platform fix (post-POC):** When the drift signal is deletion
> (`Synced=False / ReconcileError`), the approval workflow should delete the composed
> MRs before flipping management policy, so the composition always recreates from a
> clean state. Tracked in F-01.

---

### LIM-03 — GCP / GCS: Requester Pays — Permanent Observation Failure

| | |
|---|---|
| **Cloud Provider** | GCP |
| **Service** | Cloud Storage |
| **MR Kind** | `Bucket` (`storage.gcp.upbound.io`) |
| **Drift Signal** | Signal 2 — `Synced=False / ReconcileError` (observe fails with HTTP 400) |
| **Reconcilable** | No |
| **Source** | MCUCP-181 (drift-reconcile-sim) |

**Scenario**

Requester Pays is enabled on a GCS bucket externally (GCP Console or `gcloud`). Once enabled,
GCP requires a `userProject` query parameter on every subsequent API call — including `GET`
(observe), `PATCH` (update), and `DELETE`.

**Error**

```
observe failed: failed to observe the resource: [{0 Error when reading or editing Storage Bucket
"<name>": googleapi: Error 400: Bucket is a requester pays bucket but no user project provided.,
required  []}]
```

The MR enters `Synced=False, reason=ReconcileError` on **observation itself** — before any drift
recovery is attempted. This persists indefinitely.

**Why**

The Crossplane GCP provider (`provider-upjet-gcp`) does not forward a `userProject` parameter on
any API call. Once Requester Pays is enabled, every GCP API call for that bucket fails with HTTP
400. Crossplane cannot observe, update, or delete the bucket.

Unlike recovery failures (where detection works but restoration fails), this is a complete loss
of provider access to the resource. The bucket becomes permanently unmanageable from Crossplane.

**Operator Action**

1. **Disable Requester Pays in GCP Console or via `gcloud`:**
   ```bash
   gcloud storage buckets update gs://<bucket-name> --no-requester-pays
   ```
2. Once disabled, Crossplane observation resumes automatically on the next reconcile cycle.
3. If drift protection (Observe mode) was active, `managementPolicies` may need to be reset
   manually if the MR was left in an inconsistent state.

---

## Test Procedure

New drift scenarios are tested using the **drift-reconcile-sim** tool (`backend/drift-sim`).
The tool automates provisioning, mutation, drift detection, reconciliation, and teardown
end-to-end. See `backend/drift-sim/README.md` for the full flag and config reference.

### 1. Prerequisites

- `kubectl` configured against the target cluster
- GCP Application Default Credentials: `gcloud auth application-default login`
- An XR fixture for the resource under test in `crossplane/examples/`

### 2. Create or update a suite config

Add a new YAML file under `backend/drift-sim/config/`, or add attributes to an existing one.
Each entry under `attributes:` becomes one test run:

```yaml
name: My Service
resource:
  kind: MyResource
  mr_gvr: <group>/<version>/<plural>
  xr_gvr: platform.example.io/v1alpha1/<xr-plural>
  xr_fixture: crossplane/examples/<service>/xr-drift-test.yaml
  provision_timeout: 5m

attributes:
  - id: my-field
    adapter_field: fieldName
    type: enum          # string | enum | boolean | integer
    test_values: ["VALUE_A"]
    reversible: false   # false = reprovision after this test
```

Set `reversible: false` for any mutation expected to be non-recoverable or that permanently
alters the resource state.

### 3. Build the binary

```bash
cd backend/drift-sim
go build -o /tmp/drift-sim ./cmd/drift-sim
```

### 4. Run the suite

Run from the **repository root** so that `xr_fixture` paths resolve correctly:

```bash
# Single suite
/tmp/drift-sim --config backend/drift-sim/config/<service>.yaml --project <gcp-project-id>

# Multiple suites via plan
/tmp/drift-sim --plan backend/drift-sim/config/plan.yaml --project <gcp-project-id>
```

The tool writes a timestamped Markdown report to `output/<config>-<timestamp>.md`.

### 5. Review the output report

Each attribute test shows one of:

| Outcome | Meaning |
|---------|---------|
| `reconciled` | Drift detected and Crossplane reconciled it back — recoverable |
| `failed` | Drift detected but Crossplane could not reconcile — non-recoverable |
| `drift_not_detected` | Mutation applied but no drift signal fired within the timeout |
| `error` | GCP API rejected the mutation itself |

For `failed` outcomes, the exact cloud provider error is in the **Error / Detail** block of
the report.

### 6. Record findings in this document

| Result | Action |
|--------|--------|
| `reconciled` | Scenario is recoverable — no entry needed here |
| `failed` | Add a new `LIM-NN` entry under **Confirmed Findings** with the error, root cause, and operator action |
| `drift_not_detected` | Investigate root cause; if it is a platform-level limitation (e.g. late initialization), add a `DET-NN` entry under **Detection Limitations** |
| `error` | If the GCP API permanently rejects the mutation (immutable field), note it in the suite config comment — no doc entry needed unless there is operator impact |
