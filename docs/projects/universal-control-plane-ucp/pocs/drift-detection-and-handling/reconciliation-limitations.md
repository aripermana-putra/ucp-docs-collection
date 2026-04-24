---
title: "Drift Reconciliation Limitations"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Drift Reconciliation Limitations

This document lists drift scenarios where the platform **cannot automatically reconcile**
the cloud resource back to its desired state. Detection still works in all cases — the
`DriftApprovalWorkflow` is started, the operator is notified, and approval can be given.
The failure occurs in the recovery phase: after approval, Crossplane attempts to patch the
resource but the cloud API rejects the change because the field is immutable or the
operation is not supported.

**What the platform does in all failure cases:**

1. Drift detected → `DriftApprovalWorkflow` started, notification sent
2. Operator approves
3. Platform flips `managementPolicies` to `[Create, Observe, Update]`
4. Crossplane attempts to reconcile → GCP API rejects → `Synced=False / ReconcileError`
5. `WaitMRReadyActivity` detects persistent `ReconcileError` after the grace period
   and fails the workflow with `RECONCILE_FAILED`
6. Platform always flips `managementPolicies` back to `[Observe]` (Step 3c guarantee —
   the MR is never left in full management mode)

The resource remains in a drifted state. Manual intervention is required.

---

## Scenario 1 — GCP / Cloud SQL: Machine Type (Tier) Change to Incompatible Target

| Field | Value |
|-------|-------|
| **Cloud Provider** | GCP |
| **Service** | Cloud SQL (DatabaseInstance) |
| **MR Kind** | `DatabaseInstance` (`sql.gcp.upbound.io`) |
| **Drift Signal** | Signal 1 — field diff (`settings.tier`) |
| **Reconcilable** | No |

### What Happens

The Cloud SQL instance's machine type (expressed as `settings.tier`, e.g. `db-n1-standard-1`,
`db-custom-2-7680`) is changed directly in the GCP Console to a type that is incompatible
with the current instance configuration — for example, switching between shared-core
and dedicated-core tiers, or specifying a tier that does not exist in the region.

The drift watcher detects the mismatch between `spec.forProvider.settings.tier` and
`status.atProvider.settings.tier` and fires a `DriftApprovalWorkflow`.

After approval, Crossplane tries to PATCH the instance. The GCP Cloud SQL API rejects
the request with an error such as:

```
googleapi: Error 400: Invalid request: Invalid tier.
```

The instance stays with the externally set tier. `Synced=False / ReconcileError` persists
beyond the 2-minute grace period. `WaitMRReadyActivity` fails the workflow.

> **Note:** Changing between compatible tiers of the same class (e.g. `db-n1-standard-1`
> → `db-n1-standard-2`) IS reconcilable by the platform — this scenario only applies
> when the target tier is invalid or the change crosses an incompatible tier class boundary.

### Root Cause

GCP Cloud SQL validates tier changes at the API level. Not all tier transitions are
allowed on a live instance without recreation. The Crossplane provider sends the desired
tier as a PATCH and relies on the GCP API to accept or reject it — there is no client-side
tier compatibility check.

### Suggestion

1. Determine the correct, compatible target tier in GCP documentation.
2. If the GCP console change was accidental: revert the tier in GCP Console manually
   to match the desired state in `spec.forProvider`. Crossplane will detect the
   restoration on the next Observe() cycle and clear the `ReconcileError`.
3. If the tier change is intentional and valid: update `spec.forProvider.settings.tier`
   in the XDatabase claim to match the new tier. Crossplane will accept the change
   as no-diff and clear the error.
4. For incompatible tier class migrations (shared-core → dedicated-core), a new
   instance must be created, data migrated, and the old instance decommissioned.
   Update the XDatabase desired state to reflect the new instance configuration.

---

## Scenario 2 — GCP / GKE: NodePool Machine Type Change

| Field | Value |
|-------|-------|
| **Cloud Provider** | GCP |
| **Service** | GKE (NodePool) |
| **MR Kind** | `NodePool` (`container.gcp.upbound.io`) |
| **Drift Signal** | Signal 1 — field diff (`nodeConfig.machineType`) |
| **Reconcilable** | No |

### What Happens

The node pool's machine type (`spec.forProvider.nodeConfig.machineType`, e.g.
`n2-standard-4`) is externally modified. Since GKE does not expose a direct "change
machine type" API on an existing node pool, any GCP-side mutation to this field typically
reflects a recreation done manually outside of Crossplane (e.g. via `gcloud container
node-pools create/delete`).

The drift watcher detects `nodeConfig.machineType` mismatch and fires a
`DriftApprovalWorkflow`.

After approval, Crossplane attempts to PATCH the NodePool. The GKE API rejects it:

```
googleapi: Error 400: The resource "nodePool" could not be updated because
it contains immutable field: nodeConfig.machineType
```

`Synced=False / ReconcileError` persists. `WaitMRReadyActivity` fails the workflow.

### Root Cause

`nodeConfig.machineType` is immutable on a GKE NodePool after creation. The GKE API
does not support in-place machine type changes. A new node pool must be created with
the desired machine type, workloads must be migrated, and the old pool must be deleted.
Crossplane has no mechanism to perform this multi-step node pool rotation automatically.

### Suggestion

1. **If the change was accidental:** The original node pool was likely deleted and a
   new one created outside of Crossplane. The desired state in the XR still points to
   the old spec. Update `spec.forProvider.nodeConfig.machineType` in the XKubernetesCluster
   claim to match the currently running node pool's machine type to clear the drift.
2. **If the change is intentional:** Perform a managed node pool rotation:
   a. Add a new NodePool to the XR composition with the desired machine type.
   b. Drain and cordon the old node pool.
   c. Remove the old node pool from the XR composition.
   d. Let Crossplane delete the old MR.

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
