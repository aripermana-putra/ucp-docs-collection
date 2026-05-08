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
| **Confirmed Examples** | GCS `uniformBucketLevelAccess: false`, GCS `defaultEventBasedHold: false` |
| **Detectable** | No |
| **Source** | MCUCP-181 (drift-reconcile-sim) |

**Scenario**

A GCS bucket has `uniformBucketLevelAccess: false` (or `defaultEventBasedHold: false`) as the
desired state in `spec.forProvider`. Someone enables it externally (GCP Console or `gcloud`).
Drift is expected to be detected via Signal 1 (forProvider vs atProvider diff:
`desired=false, actual=true`).

**What actually happens**

Crossplane's `Observe()` cycle runs late initialization on every reconcile loop. For any field in
`spec.forProvider` that equals the Go zero value (`false` for bool, `0` for int, `""` for string),
upjet treats it as "unset" and copies the observed `atProvider` value back into `forProvider`.

After the external mutation sets the field to `true` (e.g. `uniformBucketLevelAccess: true`):

1. `atProvider.<field> = true`
2. Late init fires: `forProvider.<field>` was `false` (zero value) → overwritten to `true`
3. Diff: `forProvider=true, atProvider=true` → no drift detected

The correction closes in seconds — before the drift scan has a chance to see the mismatch.

**Confirmed instances** (from drift-reconcile-sim runs, MCUCP-181):

| Field | GVR | Desired | Mutation | Result |
|-------|-----|---------|----------|--------|
| `uniformBucketLevelAccess` | `storage.gcp.upbound.io/v1beta2/buckets` | `false` | set → `true` | `drift_not_detected` |
| `defaultEventBasedHold` | `storage.gcp.upbound.io/v1beta2/buckets` | `false` | set → `true` | `drift_not_detected` |

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

## Test Coverage by Resource Type

This section documents which fields are included in the drift-reconcile-sim test suite and why
certain fields are intentionally excluded. Config files live in `backend/drift-sim/config/`.

### Exclusion Reason Codes

| Code | Meaning |
|------|---------|
| **DET-01** | Desired state is a Go zero value (`false`, `0`, `""`). Upjet late-init overwrites `forProvider` before the drift scan fires — the drift is undetectable in Observe mode. See [DET-01](#det-01--upjet-late-initialization-overwrites-boolean-zero-value-desired-state). |
| **IMMUTABLE** | GCP permanently rejects the mutation. Tested as `error` outcome — confirms the constraint is documented. |
| **NO-BASELINE** | Field is not set by the composition, so it is absent from `forProvider`. The drift detector (`diffMaps`) only iterates keys present in `forProvider`; changes to absent fields are invisible to Signal 1. |
| **ARRAY-TYPE** | The adapter's `buildBody` helper constructs scalar dot-notation bodies only. Arrays and multi-value nested objects require custom adapter logic not yet implemented. |
| **ENDPOINT** | Mutation requires a separate GCP API endpoint not yet implemented in the current adapter (e.g. a dedicated POST endpoint instead of the general PATCH/PUT). These are real-world drift scenarios — adapter extension is required to test them, but the drift is detectable once the adapter sends the mutation. |
| **GCP-CONSTRAINT** | GCP restriction makes the test unsafe, permanently destructive, or inapplicable for the test resource configuration (e.g. feature only valid for a specific region type). |
| **NOT-API-FIELD** | Field exists in the Crossplane CRD but is not a GCP API resource field — it controls Crossplane's own behavior (e.g. `forceDestroy`, `deletionProtection`, `*Ref`, `*Selector`). |

---

### GCP / Cloud Storage — `Bucket` (`storage.gcp.upbound.io/v1beta2`)

Config: `backend/drift-sim/config/gcs.yaml`
Fixture: `crossplane/examples/objectstorage/gcs/xobjectstorage-drift-test.yaml`

**Tested:**

| Attribute | Field | Expected Outcome |
|-----------|-------|-----------------|
| `bucket-location` | `location` | `error` — immutable after creation |
| `bucket-storage-class` | `storageClass` (NEARLINE, COLDLINE, ARCHIVE) | `reconciled` |
| `bucket-public-access-prevention` | `publicAccessPrevention` → `inherited` | `reconciled` |
| `bucket-uniform-bucket-level-access` | `uniformBucketLevelAccess` → `false` | `reconciled` — fixture sets desired=`true`, not DET-01 |
| `bucket-versioning` | `versioning[0].enabled` → `true` | `reconciled` — nested bool inside explicitly-set object, not DET-01 |
| `bucket-soft-delete-retention-decrease` | `softDeletePolicy[0].retentionDurationSeconds` → `0` | `reconciled` |
| `bucket-soft-delete-retention-increase` | `softDeletePolicy[0].retentionDurationSeconds` scale_up (7d → 14d) | `reconciled` |
| `bucket-default-event-based-hold` | `defaultEventBasedHold` → `true` | `drift_not_detected` — **DET-01** regression test |

**Not tested:**

| Field | Code | Note |
|-------|------|------|
| `project` | IMMUTABLE | Buckets cannot be moved between GCP projects; field is set at creation and never changes |
| `labels` | NO-BASELINE | Externally-added label keys don't exist in `forProvider.labels`; `diffMaps` only checks keys already in `forProvider` — additions are invisible |
| `autoclass.enabled`, `autoclass.terminalStorageClass` | NO-BASELINE + DET-01 | Not set in composition; autoclass is disabled by default (desired=`false`); `terminalStorageClass` is only meaningful when `enabled=true` |
| `encryption.defaultKmsKeyName` | NO-BASELINE + DET-01 | Not set in composition; desired=`""` (empty string zero value) |
| `retentionPolicy.retentionPeriod` | NO-BASELINE | Not set in composition; externally-set retention period is invisible to `diffMaps` |
| `retentionPolicy.isLocked` | DET-01 + GCP-CONSTRAINT | Desired=`false` (zero value); also irreversible — once locked the retention policy can never be shortened or removed |
| `cors` | ARRAY-TYPE | Multi-value nested array — `buildBody` cannot construct it |
| `lifecycleRule` | ARRAY-TYPE | Multi-value nested array — `buildBody` cannot construct it |
| `customPlacementConfig.dataLocations` | GCP-CONSTRAINT + ARRAY-TYPE | Only applies to dual-region buckets; test bucket uses `US` (multi-region); `dataLocations` is also an array |
| `rpo` | GCP-CONSTRAINT | `ASYNC_TURBO` is only valid for dual-region buckets; test bucket uses `US` (multi-region) |
| `website.mainPageSuffix`, `website.notFoundPage` | DET-01 | Desired=`""` (string zero value) |
| `logging.logBucket`, `logging.logObjectPrefix` | NO-BASELINE + DET-01 | Not set in composition; desired=`""` (string zero value) |
| `requesterPays` | DET-01 + GCP-CONSTRAINT | Desired=`false` (zero value); also enabling it permanently breaks Crossplane observation (see LIM-03) |
| `enableObjectRetention` | DET-01 + GCP-CONSTRAINT | Desired=`false` (zero value); also irreversible — once enabled on a bucket it cannot be disabled |
| `uniformBucketLevelAccess` (desired=`false` scenario) | DET-01 | The reverse scenario (desired=`false`, externally enabled) is undetectable; the current fixture avoids this by setting desired=`true` |
| `forceDestroy` | NOT-API-FIELD | Controls Crossplane deletion behaviour only |

---

### GCP / Cloud SQL — `DatabaseInstance` (`sql.gcp.upbound.io/v1beta2`)

Configs: `backend/drift-sim/config/cloudsql-a.yaml`, `cloudsql-b.yaml`
Fixtures: `crossplane/examples/dbaas/cloudsql/xdatabase-drift-test.yaml` (Suite A),
`crossplane/examples/dbaas/cloudsql/xdatabase-drift-test-b.yaml` (Suite B)

Composition sets explicitly: `databaseVersion`, `project`, `region`, `deletionProtection: false`
(TF-only flag), `settings.tier`, `settings.diskSize`, `settings.diskType: PD_SSD`,
`settings.diskAutoresize: true`, `settings.availabilityType: ZONAL`,
`settings.deletionProtectionEnabled: false`, `settings.userLabels.ucp-managed: "true"`,
`settings.ipConfiguration.ipv4Enabled`.
All other fields are late-initialized from GCP defaults after first provision.

**Tested:**

| Attribute | Field | Actual Outcome |
|-----------|-------|----------------|
| `disk-size-increase` | `settings[0].diskSize` scale_up (20 → 40 GB) | ❌ `failed` — drift detected; Crossplane refuses ForceNew: "cannot change disk_size from 40 to 20" — **LIM-01** |
| `disk-size-decrease` | `settings[0].diskSize` scale_down (20 → 10 GB) | ⚠️ `error` — GCP 400: "The disk size cannot decrease" (mutation rejected at API level) |
| `db-version-upgrade` | `databaseVersion` → `POSTGRES_16` | ⚠️ `error` — adapter timeout (stale result; 30 min cap hit; fixed to 60 min but not re-run; expected outcome ❌ `failed` — ForceNew same as disk-size-increase) |
| `db-version-downgrade` | `databaseVersion` → `POSTGRES_13` | ⚠️ `error` — GCP 400: "Not allowed to do major version upgrade from POSTGRES_15 to POSTGRES_13" |
| `availability-type` → REGIONAL | `settings[0].availabilityType` → `REGIONAL` | ✅ `reconciled` — HA transition takes 12–20 min |
| `availability-type` → ZONAL | `settings[0].availabilityType` → `ZONAL` | 🔇 `drift_not_detected` — desired=`ZONAL`; mutation matches desired; no diff |
| `settings-tier` | `settings[0].tier` → `db-g1-small` | ✅ `reconciled` — instance restart required (~12 min) |
| `disk-autoresize` | `settings[0].diskAutoresize` → `false` | ✅ `reconciled` — composition sets desired=`true`, not DET-01 |
| `region-change` | `region` → `us-east1` | 🔇 `drift_not_detected` — GCP accepts PATCH without error but silently ignores the immutable field; `atProvider.region` unchanged → no diff |
| `instance-type-change` | `instanceType` → `READ_REPLICA_INSTANCE` | ⚠️ `error` — GCP 400: "instanceType cannot be updated" |
| `disk-type-change` | `settings[0].diskType` → `PD_HDD` | ⚠️ `error` — GCP 400: "Storage type cannot be changed" |
| `deletion-protection-enable` | `settings[0].deletionProtectionEnabled` → `true` | ✅ `reconciled` — composition sets desired=`false`; drift detected via Signal 1; Crossplane restores |
| `user-label-value-change` | `settings[0].userLabels.ucp-managed` → `"modified"` | ✅ `reconciled` — value-change on an existing key is detectable |
| `instance-deletion` | delete underlying GCP resource | ✅ `reconciled` — Signal 2 (`ReconcileError`: resource not found); FullPolicies recreates instance |

> **Note on `region-change`:** Unlike `instance-type-change` and `disk-type-change` which GCP
> rejects with HTTP 400, the `region` field is accepted by the SQL Admin API with no error but
> the value is not changed. This silent-accept-and-ignore behavior means the platform sees no
> mutation error and no state change — the field appears consistent even though the desired
> mutation was effectively a no-op.

> **Note on `deletionProtection` vs `settings.deletionProtectionEnabled`:** The top-level
> `deletionProtection` field in provider-upjet-gcp is a **Terraform client-side flag only** —
> it prevents `terraform destroy` but has no GCP API representation and is never reflected in
> `atProvider`. The actual GCP deletion protection field is `settings.deletionProtectionEnabled`.
> Use this path for `at_provider_path`, `crossplane_path`, and `adapter_field` in test configs.

**Not tested:**

| Field | Code | Note |
|-------|------|------|
| **Top-level** | | |
| `project` | IMMUTABLE | Composition sets it at creation; GCP does not allow moving instances between projects |
| `encryptionKeyName` | NO-BASELINE + GCP-CONSTRAINT | Not set in composition; CMEK is immutable after creation |
| `masterInstanceName` | NOT-API-FIELD | Read replicas only; not applicable to primary instances |
| `replicaConfiguration` | NOT-API-FIELD | Read replicas only |
| `onPremisesConfiguration` | NOT-API-FIELD | External server migration only |
| `maintenanceVersion` | GCP-CONSTRAINT | Minor version selection during maintenance windows; not directly PATCH-able via Admin API |
| **settings — mutable scalars** | | |
| `settings.activationPolicy` | NO-BASELINE | Not set in composition; GCP default "ALWAYS" is late-init'd (non-zero); a value-change test is possible but low priority |
| `settings.storageAutoResizeLimit` | NO-BASELINE | Not set in composition; GCP default `0` after late-init → DET-01 |
| `settings.connectorEnforcement` | NO-BASELINE | Not set in composition; GCP default "NOT_REQUIRED" after late-init |
| **settings — immutable after creation** | | |
| `settings.collation` | NO-BASELINE + GCP-CONSTRAINT | Not set in composition; immutable after instance creation |
| `settings.timeZone` | NOT-API-FIELD + GCP-CONSTRAINT | SQL Server only; immutable after creation |
| `settings.pricingPlan` | NO-BASELINE + GCP-CONSTRAINT | Only `PER_USE` valid for second-gen instances; effectively read-only |
| `settings.replicationType` | NOT-API-FIELD | First-generation instances only; deprecated |
| **settings — complex objects** | | |
| `settings.locationPreference` | NO-BASELINE | Not set in composition; `zone` and `secondaryZone` are derived from `region` |
| `settings.backupConfiguration` | NO-BASELINE | Not set in composition; complex nested object with multiple sub-fields |
| `settings.maintenanceWindow` | NO-BASELINE | Not set in composition |
| `settings.insightsConfig` | NO-BASELINE | Not set in composition |
| `settings.passwordValidationPolicy` | NO-BASELINE | Not set in composition |
| `settings.dataCacheConfig` | NO-BASELINE + GCP-CONSTRAINT | Enterprise Plus tier only |
| `settings.activeDirectoryConfig` | NOT-API-FIELD | SQL Server only |
| `settings.sqlServerAuditConfig` | NOT-API-FIELD | SQL Server only |
| **settings — arrays** | | |
| `settings.databaseFlags` | ARRAY-TYPE | Key-value flag array — `buildBody` cannot construct it |
| **settings.ipConfiguration — sub-fields** | | |
| `settings.ipConfiguration.ipv4Enabled` → `false` | GCP-CONSTRAINT | Disabling public IP without a private network configured causes GCP 400: "At least one of Public IP or Private IP must be enabled"; test fixture has no private network |
| `settings.ipConfiguration.authorizedNetworks` | ARRAY-TYPE | CIDR block array |
| `settings.ipConfiguration.requireSsl` | DET-01 | Desired=`false` (zero value); deprecated in favour of `sslMode` |
| `settings.ipConfiguration.sslMode` | NO-BASELINE | Not set in composition; default empty string → DET-01 |
| `settings.ipConfiguration.privateNetwork` | NO-BASELINE + GCP-CONSTRAINT | Not set in composition; VPC peering configuration; largely immutable after creation |
| `settings.ipConfiguration.enablePrivatePathForGoogleCloudServices` | DET-01 | Desired=`false` (zero value) |
| `settings.ipConfiguration.allocatedIpRange` | NO-BASELINE | Not set in composition |
| **userLabels — label-addition scenario** | | |
| `settings.userLabels` (new key added externally) | NO-BASELINE | `diffMaps` only checks keys present in `forProvider`; adding a brand-new label key externally is invisible to Signal 1 |
| **Crossplane control fields** | | |
| `deletionProtection` | NOT-API-FIELD | TF client-side deletion guard — not a GCP API field; never reflected in `atProvider`; use `settings.deletionProtectionEnabled` instead |
| `rootPassword` | GCP-CONSTRAINT | Mutating root credentials during a live test risks breaking active connections |
| `*Ref`, `*Selector` fields | NOT-API-FIELD | Crossplane reference resolution — not GCP API fields |

---

### GCP / GKE — `Cluster` (`container.gcp.upbound.io/v1beta2`)

Config: `backend/drift-sim/config/gke-cluster.yaml`
Fixture: `crossplane/examples/kubernetes/gke/xkubernetescluster-drift-test.yaml`

Composition sets explicitly: `location`, `releaseChannel.channel`, `loggingService: "none"`,
`monitoringService: "none"`, `deletionProtection: false`. All other fields are late-initialized
from GCP defaults.

**Tested:**

| Attribute | Field | Expected Outcome |
|-----------|-------|-----------------|
| `cluster-network` | `network` | `error` — immutable |
| `cluster-location` | `location` | `error` — immutable |
| `cluster-ipv4-cidr` | `clusterIpv4Cidr` | `error` — immutable |
| `cluster-subnetwork` | `subnetwork` | `error` — immutable |
| `cluster-release-channel` | `releaseChannel[0].channel` → `RAPID` | `reconciled` (no reprovision — channel downgrade not supported) |
| `cluster-logging-service` | `loggingService` → `logging.googleapis.com/kubernetes` | `reconciled` — desired=`"none"` (non-empty), not DET-01 |
| `cluster-monitoring-service` | `monitoringService` → `monitoring.googleapis.com/kubernetes` | `reconciled` — desired=`"none"` (non-empty), not DET-01 |
| `cluster-pd-csi-driver` | `addonsConfig[0].gcePersistentDiskCsiDriverConfig[0].enabled` → `false` | `reconciled` — enabled by default on GKE 1.14+, desired=`true` late-init, not DET-01 |
| `cluster-filestore-csi-driver` | `addonsConfig[0].gcpFilestoreCsiDriverConfig[0].enabled` → `false` | `reconciled` — enabled by default on GKE 1.21+, desired=`true` late-init, not DET-01 |

**Not tested:**

| Field | Code | Note |
|-------|------|------|
| `enableIntranodeVisibility` | DET-01 | Disabled by default; desired=`false` after late-init |
| `enableShieldedNodes` | DET-01 | Disabled by default; desired=`false` after late-init |
| `enableLegacyAbac` | DET-01 | Disabled by default; desired=`false` after late-init |
| `networkPolicy.enabled` | DET-01 | Disabled by default; desired=`false` after late-init |
| `costManagementConfig.enabled` | DET-01 | Disabled by default; desired=`false` after late-init |
| `identityServiceConfig.enabled` | DET-01 | Disabled by default; desired=`false` after late-init |
| `addonsConfig.horizontalPodAutoscaling.disabled` | DET-01 | `disabled: false` (HPA enabled) is the zero value — externally disabling is undetectable |
| `addonsConfig.httpLoadBalancing.disabled` | DET-01 | Same as above |
| `addonsConfig.networkPolicyConfig.disabled` | DET-01 | Same pattern |
| `addonsConfig.dnsCacheConfig.enabled` | DET-01 | Disabled by default; desired=`false` |
| `masterAuthorizedNetworksConfig.cidrBlocks` | ARRAY-TYPE | CIDR block array |
| `loggingConfig.enableComponents` | ARRAY-TYPE | Component name array |
| `monitoringConfig.enableComponents` | ARRAY-TYPE | Component name array |
| `nodeLocations` | ARRAY-TYPE | Zone string array |
| `workloadIdentityConfig` | NO-BASELINE | Not set in composition; complex object |
| `maintenancePolicy` | NO-BASELINE | Not set in composition |
| `resourceLabels` | NO-BASELINE | Not set in composition; externally-added keys invisible to `diffMaps` |
| `networkingMode`, `datapathProvider` | NO-BASELINE | Not set in composition; likely immutable after creation |
| `ipAllocationPolicy.*` | NO-BASELINE + IMMUTABLE | Not set in composition; most sub-fields are immutable after creation |
| `deletionProtection` | NOT-API-FIELD | Controls Crossplane deletion behaviour only |
| `*Ref`, `*Selector` fields | NOT-API-FIELD | Crossplane reference resolution — not GCP API fields |

---

### GCP / GKE — `NodePool` (`container.gcp.upbound.io/v1beta2`)

Config: `backend/drift-sim/config/gke-nodepool.yaml`
Fixture: `crossplane/examples/kubernetes/gke/xkubernetescluster-drift-test.yaml` (same as Cluster)

Composition sets explicitly: `location`, `initialNodeCount`, `autoscaling.minNodeCount`,
`autoscaling.maxNodeCount`, `management.autoRepair: true`, `management.autoUpgrade: true`,
`nodeConfig.machineType`, `nodeConfig.diskSizeGb`, `nodeConfig.oauthScopes`,
`nodeConfig.shieldedInstanceConfig.enableSecureBoot: true`.

**Tested:**

| Attribute | Field | Expected Outcome |
|-----------|-------|-----------------|
| `nodepool-machine-type` | `nodeConfig[0].machineType` | `error` — nodeConfig is immutable after pool creation |
| `nodepool-disk-type` | `nodeConfig[0].diskType` (pd-ssd, pd-balanced) | `error` — nodeConfig is immutable after pool creation |
| `nodepool-image-type` | `nodeConfig[0].imageType` (COS_CONTAINERD, UBUNTU_CONTAINERD) | `reconciled` |
| `nodepool-max-surge` | `upgradeSettings[0].maxSurge` scale_up (1 → 2) | `reconciled` — mutable via `updateNodePool` PUT |

**Not tested:**

| Field | Code | Note |
|-------|------|------|
| `autoscaling.minNodeCount`, `autoscaling.maxNodeCount` | ENDPOINT | Require `setAutoscaling` POST endpoint; current adapter only implements `updateNodePool` PUT |
| `management.autoRepair`, `management.autoUpgrade` | ENDPOINT | Require `setManagement` POST endpoint |
| `upgradeSettings.maxUnavailable` | DET-01 | Late-initialized to `0` (integer zero value) |
| `upgradeSettings.strategy` → `BLUE_GREEN` | GCP-CONSTRAINT | Requires `blueGreenSettings` to be populated simultaneously — not expressible as a single scalar mutation |
| `nodeConfig.diskSizeGb` | ENDPOINT | Not included in `updateNodePool` PUT body; mutation would be silently ignored, producing a false `drift_not_detected` |
| `nodeConfig.labels` | NO-BASELINE | Not set in composition; externally-added label keys invisible to `diffMaps` |
| `nodeConfig.taint` | ARRAY-TYPE | Array of `{effect, key, value}` objects |
| `nodeConfig.preemptible`, `nodeConfig.spot` | DET-01 | Desired=`false` (zero value); also both are immutable after pool creation |
| `nodeConfig.accelerators` | ARRAY-TYPE + NO-BASELINE | GPU config array; not set in composition |
| `nodeConfig.kubeletConfig`, `nodeConfig.linuxNodeConfig` | NO-BASELINE | Not set in composition |
| `nodeConfig.shieldedInstanceConfig.enableIntegrityMonitoring` | DET-01 | Desired=`false` after late-init |
| `networkConfig.enablePrivateNodes` | DET-01 | Desired=`false` after late-init |
| `nodeLocations` | ARRAY-TYPE | Zone string array |
| `version` | GCP-CONSTRAINT | Node version upgrades are slow and depend on available GKE minor versions; tested manually |
| `placementPolicy` | NO-BASELINE + GCP-CONSTRAINT | Not set in composition; `COMPACT` policy requires homogeneous machine types |
| `*Ref`, `*Selector`, `clusterRef` | NOT-API-FIELD | Crossplane reference resolution fields |

---

### GCP / Compute Engine — `Instance` (`compute.gcp.upbound.io/v1beta1`)

Config: `backend/drift-sim/config/gce.yaml`
Fixture: `crossplane/examples/compute/gcp/xcomputeinstance-drift-test.yaml`

Composition sets explicitly: `machineType` (from size/machineType parameter), `zone`, `project`,
`bootDisk[0].initializeParams[0].image`, `bootDisk[0].initializeParams[0].size`,
`shieldedInstanceConfig[0].enableSecureBoot: true`, `shieldedInstanceConfig[0].enableVtpm: true`,
`shieldedInstanceConfig[0].enableIntegrityMonitoring: true`,
`networkInterface[0].network`, `networkInterface[0].accessConfig[0].networkTier: PREMIUM` (when `publicIp: true`).
All other fields are late-initialized from GCP defaults after first provision.

Adapter note: the GCE adapter implements `instances.setShieldedInstanceConfig` (PATCH). Mutations to
other field groups (`machineType`, `tags`, `labels`, `metadata`, `scheduling`) each require separate
dedicated GCP API endpoints not yet implemented.

**Tested:**

| Attribute | Field | Expected Outcome |
|-----------|-------|-----------------|
| `shielded-secure-boot` | `shieldedInstanceConfig[0].enableSecureBoot` → `false` | `reconciled` — composition sets desired=`true`, not DET-01; GCP records change in API even before restart |
| `shielded-integrity-monitoring` | `shieldedInstanceConfig[0].enableIntegrityMonitoring` → `false` | `reconciled` — can be toggled on running instance without restart |
| `machine-type-change` | `machineType` → `e2-small` | `reconciled` — adapter stops instance, calls `setMachineType`, starts instance; Crossplane restores `e2-micro` |
| `boot-disk-size-increase` | `bootDisk[0].initializeParams[0].size` scale_up (20 GB → 40 GB) | `failed` — drift is detected and reconcile is attempted but GCP permanently rejects disk shrink (same pattern as LIM-01) |

**Not tested:**

| Field | Code | Note |
|-------|------|------|
| **shieldedInstanceConfig** | | |
| `shieldedInstanceConfig[0].enableVtpm` | GCP-CONSTRAINT | GCP rejects disabling vTPM while `enableSecureBoot=true`; would require disabling Secure Boot first in the same request — not expressible as a single-field mutation |
| **bootDisk** | | |
| `bootDisk[0].initializeParams[0].image` | IMMUTABLE | Boot disk image is immutable after creation; any change requires reprovisioning |
| **machineType and zone** | | |
| `zone` | IMMUTABLE | Zone is immutable after instance creation |
| `project` | IMMUTABLE | Project is immutable after instance creation |
| **networkInterface — immutable after creation** | | |
| `networkInterface[0].network` | IMMUTABLE | Network is immutable after creation |
| `networkInterface[0].subnetwork` | IMMUTABLE | Subnetwork is immutable after creation |
| `networkInterface[0].networkIp` | IMMUTABLE | Internal IP is immutable after creation (though ephemeral) |
| `networkInterface[0].accessConfig[0].networkTier` | ENDPOINT | Changing network tier requires deleting and re-adding the access config via `accessConfigs.delete` + `accessConfigs.insert` — not a single PATCH |
| `networkInterface[0].aliasIpRanges` | ARRAY-TYPE + ENDPOINT | Array of IP ranges; uses `updateNetworkInterface` endpoint |
| **fields needing dedicated endpoints** | | |
| `tags` | ENDPOINT | `instances.setTags` (POST); requires current tag fingerprint |
| `labels` | NO-BASELINE + ENDPOINT | Not set in composition; `instances.setLabels` (POST) requires current label fingerprint |
| `metadata` | NO-BASELINE + ENDPOINT | Not set in composition; `instances.setMetadata` (POST) requires current metadata fingerprint; array of key-value items |
| `scheduling.automaticRestart` | NO-BASELINE + ENDPOINT | Not set in composition; `instances.setScheduling` endpoint |
| `scheduling.onHostMaintenance` | NO-BASELINE + ENDPOINT | Not set in composition; `instances.setScheduling` endpoint |
| `scheduling.preemptible` | NO-BASELINE + GCP-CONSTRAINT + ENDPOINT | Not set in composition; immutable after creation (cannot change between preemptible and standard) |
| **DET-01 fields** | | |
| `canIpForward` | DET-01 + IMMUTABLE | Desired=`false` (zero value); also immutable after creation |
| `description` | DET-01 | Desired=`""` (string zero value) |
| `minCpuPlatform` | DET-01 | Desired=`""` (string zero value) |
| `hostname` | DET-01 + GCP-CONSTRAINT | Desired=`""` (zero value); custom hostname is immutable after creation |
| `advancedMachineFeatures.enableNestedVirtualization` | DET-01 + GCP-CONSTRAINT | Desired=`false`; immutable after creation |
| **complex objects and arrays** | | |
| `serviceAccount` | NO-BASELINE + ENDPOINT + ARRAY-TYPE | Not set in composition; requires `instances.setServiceAccount` endpoint |
| `guestAccelerators` | NO-BASELINE + ARRAY-TYPE + GCP-CONSTRAINT | Not set in composition; GPU config; immutable after creation |
| `attachedDisk` | NO-BASELINE + ARRAY-TYPE + ENDPOINT | Additional disks; `disks.attachDisk` / `disks.detachDisk` endpoints |
| `scratchDisk` | NO-BASELINE + GCP-CONSTRAINT + ARRAY-TYPE | Not set in composition; local SSDs are immutable after creation |
| `resourcePolicies` | NO-BASELINE + ARRAY-TYPE | Not set in composition; maintenance policies |
| `reservationAffinity` | NO-BASELINE | Not set in composition |
| `confidentialInstanceConfig.enableConfidentialCompute` | GCP-CONSTRAINT + IMMUTABLE | Immutable after creation; requires a compatible machine type |
| **Crossplane control fields** | | |
| `deletionProtection` | NOT-API-FIELD | Controls Crossplane deletion behaviour only |
| `*Ref`, `*Selector` fields | NOT-API-FIELD | Crossplane reference resolution — not GCP API fields |

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
/tmp/drift-sim --config backend/drift-sim/config/<service>.yaml

# Multiple suites via plan
/tmp/drift-sim --plan backend/drift-sim/config/plan.yaml
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
