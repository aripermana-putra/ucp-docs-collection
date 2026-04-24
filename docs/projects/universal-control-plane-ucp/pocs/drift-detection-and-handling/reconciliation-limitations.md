---
title: "Drift Reconciliation Limitations"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Drift Reconciliation Limitations

This document lists drift scenarios where the platform **cannot automatically reconcile**
the cloud resource back to its desired state. Detection still works in all cases — the
`DriftApprovalWorkflow` is started, the operator is notified, and approval can be given.
The failure occurs in the recovery phase: after approval, Crossplane attempts to patch or
recreate the resource but either the cloud API rejects the change or stale state in
`forProvider` prevents successful recreation.

**Related findings:** [F-01](poc-results.md#f-01-auto-reconciliation-is-unreliable-for-deletion-drift-on-complex-resources),
[F-07](poc-results.md#f-07-some-drifts-are-non-recoverable--crossplane-cannot-reverse-all-gcp-changes)

---

**What the platform does in all failure cases:**

1. Drift detected → `DriftApprovalWorkflow` started, notification sent
2. Operator approves
3. Platform flips `managementPolicies` to `[Create, Observe, Update]`
4. Crossplane attempts to reconcile → cloud API rejects or recreation fails
5. `WaitMRReadyActivity` detects persistent `ReconcileError` after the 2-minute grace
   period and fails the workflow early with `RECONCILE_FAILED` (non-retryable), including
   the exact GCP error message
6. Platform always flips `managementPolicies` back to `[Observe]` (Step 3c guarantee —
   the MR is never left in full management mode)

The resource remains in a drifted state. Manual intervention is required.
The exact GCP error message is visible in the Temporal UI under the failed workflow's
`WaitMRReadyActivity` event.

---

## Scenario 1 — GCP / Cloud SQL: Disk Size Reduction

| Field | Value |
|-------|-------|
| **Cloud Provider** | GCP |
| **Service** | Cloud SQL (DatabaseInstance) |
| **MR Kind** | `DatabaseInstance` (`sql.gcp.upbound.io`) |
| **Drift Signal** | Signal 1 — field diff (`settings.disk_size`) |
| **Reconcilable** | No |
| **Confirmed in** | F-07 (MCUCP-158 implementation testing) |

### What Happens

The Cloud SQL instance's disk size is manually increased in the GCP Console (e.g. 50 GB
→ 150 GB). The drift watcher detects the mismatch between `spec.forProvider.settings.disk_size`
(50) and `status.atProvider.settings.disk_size` (150) and fires a `DriftApprovalWorkflow`.

After approval, Crossplane attempts to PATCH the instance to set `disk_size` back to 50 GB.
The GCP Cloud SQL API rejects this:

```
update failed: async update failed: refuse to update the external resource because the
following update requires replacing it: cannot change the value of the argument
"settings.0.disk_size" from "150" to "50"
```

The MR condition: `Synced=False, reason=ReconcileError, type=LastAsyncOperation: AsyncUpdateFailure`.
This persists indefinitely. `WaitMRReadyActivity` hits the grace period and exits early with
`RECONCILE_FAILED`. Step 3c flips back to `[Observe]`.

### Root Cause

GCP Cloud SQL enforces that disk size can only increase, never decrease. This is a hard
constraint at the GCP API level — disk storage once allocated cannot be reclaimed on a
live instance. Crossplane has no pre-flight check for this; it sends the PATCH and relies
on the API response.

### Suggestion

**Option A — Accept the new disk size (recommended):**
Update `spec.forProvider.settings.disk_size` in the XDatabase claim to `150` (or whatever
the current GCP value is). Crossplane will see no diff and clear the `ReconcileError`. The
larger disk is billed accordingly.

**Option B — Revert in GCP (not possible):**
GCP does not allow reducing disk size on a live instance. This option is unavailable.

**Option C — Recreate the instance (data loss risk):**
If the smaller disk size is a hard requirement (e.g. cost control), a new Cloud SQL instance
must be created at the desired size, data migrated from the old instance, and the old instance
deleted. This involves downtime and should be planned as a maintenance operation.

---

## Scenario 2 — GCP / GKE: Cluster or NodePool Deleted with Stale Late-Initialized Fields

| Field | Value |
|-------|-------|
| **Cloud Provider** | GCP |
| **Service** | GKE (Cluster, NodePool) |
| **MR Kind** | `Cluster`, `NodePool` (`container.gcp.upbound.io`) |
| **Drift Signal** | Signal 2 — `Synced=False / ReconcileError` (resource not found) |
| **Reconcilable** | No (without manual intervention) |
| **Confirmed in** | F-01, F-07 (POC testing and MCUCP-158 implementation testing) |

### What Happens

A GKE Cluster or NodePool is deleted directly from the GCP Console. Crossplane detects the
deletion via `Synced=False / ReconcileError`. The drift watcher fires a `DriftApprovalWorkflow`.

After approval, the platform flips `managementPolicies` to `[Create, Observe, Update]`.
Crossplane attempts to recreate the resource using the current `spec.forProvider` state.
The recreation fails at the GCP API level due to **stale late-initialized fields** accumulated
from the original resource's lifecycle.

**Failure sequence observed during testing (GKE Cluster):**

1. Cluster recreate fails: `Error 400: Cannot specify logging_config together with logging_service`
   — the composition had `loggingService: "none"` (deprecated field); late initialization had
   also written `loggingConfig` (new field) into `forProvider` from the original cluster's
   observed state. GCP rejects requests with both field generations simultaneously.
2. After fixing the composition: Cluster recreate fails again: `Error 400: SYSTEM_COMPONENTS
   monitoring must be enabled if any monitoring is enabled` — the late-initialized
   `forProvider.monitoringConfig` from the old cluster had a component list missing
   `SYSTEM_COMPONENTS`.
3. After fixing the composition again: Cluster creates. NodePool recreate fails:
   pod secondary range `gke-ari-test-kube-cluster-1-pods-0f92ba7c` not found — the
   late-initialized `forProvider.networkConfig.podRange` referenced a range that belonged
   to the deleted cluster and no longer exists.

### Root Cause

**Late initialization** is the underlying cause. After Crossplane creates a resource, the
provider's `Observe()` cycle writes GCP's full API response back into `spec.forProvider` —
including fields the composition never declared. These are GCP-assigned values specific to
the resource instance (pod secondary range names, auto-assigned component lists, etc.).

When the resource is deleted and Crossplane attempts recreation from `forProvider`, those
field values reference state that was specific to the now-deleted resource. GCP rejects the
request because the referenced state no longer exists or because deprecated and new fields
conflict.

This is a **deletion-specific failure** — field-level drift (resource still exists) is not
affected because Crossplane sends an UPDATE to an existing resource where the late-init
values are still valid.

### Suggestion

**Option A — Delete the MR and let the composition recreate it from a clean state:**
```bash
# 1. Delete the composed MR directly — Crossplane will detect the missing MR
#    and recreate it via the composition, without the stale late-init fields
kubectl delete cluster.container.gcp.upbound.io <mr-name>

# 2. Watch Crossplane recreate the MR
kubectl get cluster.container.gcp.upbound.io -w
```
This bypasses the stale `forProvider` entirely. The composition generates a clean `forProvider`
on first create, and `Observe()` re-initializes it from the new resource's state.

**Option B — Manually clean `forProvider` before approving:**
Before approving the drift workflow, edit the MR to remove late-initialized fields that
reference the deleted resource (pod secondary range names, conflicting logging/monitoring
fields). This is error-prone and requires GKE API knowledge.

**Longer-term fix (tracked as post-POC):**
When the drift signal is `Synced=False / ReconcileError` (deletion), the approval workflow
should delete the composed MRs before flipping management policy, forcing the composition
to recreate them from scratch. See F-01 recommendation in poc-results.md.

---

## Additional Scenarios to Investigate

The following scenarios have not been tested but are expected to be irrecoverable based
on GCP API constraints. Each should be validated with a test run.

### GCP / Cloud SQL

| Scenario | Immutable Field | Expected Outcome | Notes |
|----------|----------------|-----------------|-------|
| **Database version downgrade** | `databaseVersion` (e.g. `POSTGRES_14` → `POSTGRES_13`) | `RECONCILE_FAILED` | GCP only allows major version upgrades, never downgrades. Upgrades may also require maintenance windows. |
| **Region change** | `region` | `RECONCILE_FAILED` | Cloud SQL instances are region-bound. A region change requires creating a new instance and migrating data. |
| **Instance type change** | `instanceType` (`CLOUD_SQL_INSTANCE` ↔ `READ_REPLICA_INSTANCE`) | `RECONCILE_FAILED` | Cannot promote a read replica to primary via a PATCH; requires specific promotion API. |
| **Database version upgrade** | `databaseVersion` (e.g. `POSTGRES_13` → `POSTGRES_14`) | Likely recoverable, but risky | GCP allows major version upgrades but they require instance downtime. Need to verify whether Crossplane can trigger this cleanly. |

### GCP / GKE

| Scenario | Immutable Field | Expected Outcome | Notes |
|----------|----------------|-----------------|-------|
| **Cluster network / subnetwork change** | `network`, `subnetwork` | `RECONCILE_FAILED` | Cluster VPC network is set at creation time and is immutable. |
| **Cluster location change** | `location` | `RECONCILE_FAILED` | Cluster region or zone is immutable after creation. |
| **Cluster IP range change** | `clusterIpv4Cidr`, `servicesIpv4Cidr` | `RECONCILE_FAILED` | IP ranges are allocated at creation and cannot be changed on a live cluster. |
| **NodePool machine type change** | `nodeConfig.machineType` | `RECONCILE_FAILED` | Machine type is part of the immutable `nodeConfig` block; changing it requires creating a new node pool. |
| **NodePool disk type change** | `nodeConfig.diskType` | `RECONCILE_FAILED` | Disk configuration is part of the immutable `nodeConfig` block. |
| **NodePool image type change** | `nodeConfig.imageType` | Possibly recoverable | Some image type changes (e.g. `COS_CONTAINERD` ↔ `UBUNTU_CONTAINERD`) can be applied via a node pool upgrade. Needs verification. |

### GCP / Compute Engine (VM Instance)

| Scenario | Immutable Field | Expected Outcome | Notes |
|----------|----------------|-----------------|-------|
| **Zone change** | `zone` | `RECONCILE_FAILED` | VMs are zone-bound. Moving a VM between zones requires snapshot + recreate. |
| **Boot disk type change** | `bootDisk.type` (`pd-standard` ↔ `pd-ssd`) | `RECONCILE_FAILED` | Boot disk type is set at VM creation; disk type cannot be changed in place (disk must be recreated). |
| **Machine type change** | `machineType` | Possibly recoverable | GCP allows machine type changes when the VM is stopped. Crossplane may trigger a stop→update→start cycle; needs verification on whether the provider handles this. |

### GCP / Cloud Storage

| Scenario | Immutable Field | Expected Outcome | Notes |
|----------|----------------|-----------------|-------|
| **Bucket location change** | `location` | `RECONCILE_FAILED` | Bucket location is immutable after creation. Data must be migrated to a new bucket. |
| **Storage class change on location-locked bucket** | `storageClass` | Possibly recoverable | Standard storage class changes (`STANDARD` ↔ `NEARLINE` ↔ `COLDLINE`) are supported via `UpdateBucket`. May work; needs testing. |

---

## Testing These Scenarios

For each scenario above, the test procedure follows the same pattern:

1. Apply an XR with `driftProtection: true` and let Crossplane provision the resource.
2. Confirm MR is in `managementPolicies: ["Observe"]` and `atProvider` is populated.
3. Directly mutate the immutable field in GCP Console or via `gcloud`.
4. Wait for `DriftApprovalWorkflow` to appear in Temporal UI.
5. Approve via platform UI or: `temporal workflow signal --workflow-id <id> --name approval-signal --input '{"approved":true}'`
6. Check `WaitMRReadyActivity` outcome in Temporal UI.
7. Confirm `managementPolicies` is returned to `["Observe"]` regardless of outcome.
8. Record result and GCP error message in this document.

> **Expected result for all immutable-field scenarios:** workflow ends with
> `RECONCILE_FAILED`, MR is back in `["Observe"]`, resource remains drifted.
> Manual remediation required per the suggestion in each scenario.
