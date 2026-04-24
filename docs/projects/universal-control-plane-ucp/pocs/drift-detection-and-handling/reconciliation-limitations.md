---
title: "Drift Reconciliation Limitations"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Drift Reconciliation Limitations

This is a living document. It captures every drift scenario where the platform cannot
automatically reconcile a cloud resource back to its desired state. Each entry is added
after the scenario has been tested end-to-end against the running platform.

Drift **detection** works in all cases listed here — the `DriftApprovalWorkflow` fires,
the operator is notified, and approval can be given. The limitation is in the **recovery
phase**: after approval, Crossplane attempts to restore the resource but the cloud provider
permanently rejects the change.

**What the platform guarantees even when recovery fails:**
- `WaitMRReadyActivity` detects the persistent `ReconcileError` after a 2-minute grace
  period and surfaces the exact cloud error message in the Temporal UI
- `managementPolicies` is always flipped back to `[Observe]` — the MR is never left in
  full management mode regardless of recovery outcome

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

## Candidates to Test

The following scenarios have not been tested yet. Each is expected to be non-recoverable
based on GCP API constraints. Results should be recorded here after testing.

### GCP / Cloud SQL

| ID | Scenario | Field | Expected | Notes |
|----|----------|-------|----------|-------|
| C-SQL-01 | Database version downgrade | `databaseVersion` (e.g. `POSTGRES_14` → `POSTGRES_13`) | Non-recoverable | GCP only supports in-place upgrades, never downgrades |
| C-SQL-02 | Region change | `region` | Non-recoverable | Instances are region-bound; cross-region move requires recreate + data migration |
| C-SQL-03 | Instance type change | `instanceType` (`CLOUD_SQL_INSTANCE` ↔ `READ_REPLICA_INSTANCE`) | Non-recoverable | Replica promotion has its own GCP API path; cannot be done via PATCH |
| C-SQL-04 | Database version upgrade | `databaseVersion` (e.g. `POSTGRES_13` → `POSTGRES_14`) | Possibly recoverable | GCP supports in-place upgrades but requires maintenance window; need to verify Crossplane handles this cleanly |

### GCP / GKE

| ID | Scenario | Field | Expected | Notes |
|----|----------|-------|----------|-------|
| C-GKE-01 | Cluster network or subnetwork change | `network`, `subnetwork` | Non-recoverable | Set at cluster creation, immutable afterward |
| C-GKE-02 | Cluster location change | `location` | Non-recoverable | Region/zone is immutable after creation |
| C-GKE-03 | Cluster IP range change | `clusterIpv4Cidr`, `servicesIpv4Cidr` | Non-recoverable | IP ranges are allocated at creation and cannot be changed on a live cluster |
| C-GKE-04 | NodePool machine type change | `nodeConfig.machineType` | Non-recoverable | `nodeConfig` is immutable after pool creation; changing machine type requires a new node pool |
| C-GKE-05 | NodePool disk type change | `nodeConfig.diskType` | Non-recoverable | Part of immutable `nodeConfig` block |
| C-GKE-06 | NodePool image type change | `nodeConfig.imageType` | Possibly recoverable | Some image type changes can be applied via node pool upgrade; needs verification |

### GCP / Compute Engine

| ID | Scenario | Field | Expected | Notes |
|----|----------|-------|----------|-------|
| C-GCE-01 | Zone change | `zone` | Non-recoverable | VMs are zone-bound; cross-zone move requires snapshot + recreate |
| C-GCE-02 | Boot disk type change | `bootDisk.type` (`pd-standard` ↔ `pd-ssd`) | Non-recoverable | Disk type is set at VM creation and cannot be changed in place |
| C-GCE-03 | Machine type change | `machineType` | Possibly recoverable | GCP allows this when the VM is stopped; need to verify whether the Crossplane provider handles the stop→update→start cycle |

### GCP / Cloud Storage

| ID | Scenario | Field | Expected | Notes |
|----|----------|-------|----------|-------|
| C-GCS-01 | Bucket location change | `location` | Non-recoverable | Location is immutable after bucket creation; data must be migrated to a new bucket |
| C-GCS-02 | Storage class change | `storageClass` (`STANDARD` ↔ `NEARLINE` ↔ `COLDLINE`) | Possibly recoverable | GCP supports storage class changes via `UpdateBucket`; need to verify Crossplane handles this |

---

## Test Procedure

For each candidate above:

1. Apply an XR with `driftProtection: true` and confirm the resource is provisioned and
   `managementPolicies: ["Observe"]` is set on the MR.
2. Confirm `status.atProvider` is populated (Observe() has run at least once).
3. Trigger the scenario directly in GCP Console or via `gcloud`.
4. Wait for a `DriftApprovalWorkflow` to appear in Temporal UI.
5. Approve via the platform UI or CLI:
   ```bash
   temporal workflow signal \
     --workflow-id <workflow-id> \
     --name approval-signal \
     --input '{"approved":true}'
   ```
6. Check the `WaitMRReadyActivity` outcome in Temporal UI. If it fails, copy the exact
   error message from the activity result.
7. Confirm `managementPolicies` is back to `["Observe"]` regardless of outcome.
8. Record the result, error message, root cause, and operator action in this document
   under **Confirmed Findings**, and remove the entry from **Candidates to Test**.
