---
title: "Test Cases"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Drift Detection POC — Test Cases

Test cases for all four approaches. Run each case on its respective branch.
Where the steps differ per approach, the difference is noted inline under **How To**.

**Branches:**

| Approach | Branch |
|----------|--------|
| A — Polling Watcher | `feature/drift-poc-approach-a-watcher` |
| B — WatchOperations | `feature/drift-poc-approach-b-watchoperations` |
| C — Informer Watcher | `feature/drift-poc-approach-c-informer` |
| D — Temporal Schedule | `feature/drift-poc-approach-d-temporal-schedule` |

**Result legend:** `pass` / `fail` / `skip` / `—` (not yet run)

---

## Section 1 — Setup Verification

Pre-conditions that must pass before any drift test is meaningful.

| ID | Summary | How To | Result |
|----|---------|--------|:-:|
| S-01 | MR has drift-protection label | `kubectl get <mr-resource> -A -l platform.io/drift-protection=true` — should list the expected MR | ✅ |
| S-02 | MR is in Observe mode | `kubectl get <mr-resource> <name> -o jsonpath='{.spec.managementPolicies}'` — expected: `["Observe"]` | — |
| S-03 | `atProvider` is populated | `kubectl get <mr-resource> <name> -o jsonpath='{.status.atProvider}'` — must be a non-empty object (Observe() has run at least once; wait up to 2 min after creation) | ✅ | 
| S-04 | XR is Ready | `kubectl wait xdatabase/<name> --for=condition=Ready --timeout=30m` (or equivalent for other XR types) | ✅ |
| S-05 | Watcher / trigger is running | **A/C:** `kubectl get pod -l app=drift-watcher` — Running. **B:** `kubectl get watchoperation -A` — entries present. **D:** `temporal schedule describe --schedule-id drift-scan` — Running | — |

---

## Section 2 — Core Drift Detection

### Field-level drift (Signal 1)

| ID | Summary | How To | A | B | C | D |
|----|---------|--------|:-:|:-:|:-:|:-:|
| D-01 | CloudSQL tier change detected | 1. In GCP Console → SQL → edit instance → change machine type. 2. Wait up to 2 min for provider poll (`--poll=1m`). 3. Check watcher output for `DRIFT DETECTED` with field `settings.tier`. | ✅ | — | — | — |
| D-02 | GCE instance machineType change detected | 1. Stop VM in console. 2. Edit → change machine type. 3. Start VM. 4. Wait up to 2 min. 5. Confirm `machineType` appears in drift output. | ✅ | — | — | — |
| D-03 | GKE Cluster autoscaling profile change detected | 1. GCP Console → GKE → cluster → edit → Cluster Autoscaler → change profile (BALANCED ↔ OPTIMIZE_UTILIZATION). 2. Wait up to 2 min. 3. Confirm `clusterAutoscaling.autoscalingProfile` in drift output. | ✅ | — | — | — |
| D-04 | GKE NodePool autoRepair toggle detected | 1. GCP Console → GKE → cluster → node pools → select pool → edit → disable auto-repair. 2. Wait up to 2 min. 3. Confirm `management.autoRepair` in drift output. | ✅ | — | — | — |
| D-05 | GCS Bucket storageClass change detected | 1. GCP Console → Storage → bucket → edit storage class (STANDARD ↔ NEARLINE). 2. Wait up to 2 min. 3. Confirm `storageClass` in drift output. | ✅ | — | — | — |
| D-06 | Multiple drifted fields reported in one event | Change two independent fields on the same resource simultaneously. Verify both fields appear in the same drift report under `changes (2):`. | ✅ | — | — | — |

### Resource deletion drift (Signal 2)

| ID | Summary | How To | A | B | C | D |
|----|---------|--------|:-:|:-:|:-:|:-:|
| D-07 | CloudSQL deletion detected via Synced=False | 1. Delete the CloudSQL instance from GCP Console. 2. Wait for `kubectl get databaseinstance <name> -o jsonpath='{.status.conditions}'` to show `Synced=False, reason=ReconcileError`. 3. Confirm drift output shows `.status.conditions[Synced]` drift entry. | ✅ | — | — | — |
| D-08 | Resource deletion is detected even though field diff is empty | After deletion, `atProvider` retains stale values that match `forProvider` (Signal 1 would return no diff). Verify that only the `Synced=False` signal triggers the alert, not a field diff. Check drift output: `changes (1): .status.conditions[Synced]`. | ✅ | — | — | — |
| D-09 | GKE NodePool deletion detected | Delete the node pool from GCP Console. Wait for `Synced=False`. Confirm drift fires for the NodePool MR. | ✅ | — | — | — |

---

## Section 3 — False Positive Prevention

These tests verify that the watcher does NOT fire a drift alert when nothing meaningful has changed.
After each test, run for at least 2 full poll cycles with no code changes in GCP and confirm no `DRIFT DETECTED` log entry appears.

| ID | Summary | How To | A | B | C | D |
|----|---------|--------|:-:|:-:|:-:|:-:|
| FP-01 | No false positive on empty desired slice vs nil atProvider | Composition declares `loggingConfig.enableComponents: []` (empty list); GCP omits the field in `atProvider`. Run watcher. Confirm no `DRIFT DETECTED` log appears for `enableComponents`. | — | — | — | — |
| FP-02 | No false positive on URL-normalized network fields | `forProvider.network = "default"`; GCP returns the full URL (e.g. `https://www.googleapis.com/.../networks/default`) in `atProvider`. Run watcher. Confirm no drift reported on `network`. | — | — | — | — |
| FP-03 | No false positive on `bootDisk.initializeParams.image` | GCE image family `debian-cloud/debian-12` is resolved by GCP to a versioned URL. Run watcher against a real GCE instance MR. Confirm no drift on `bootDisk[0].initializeParams[0].image`. | ✅ | N/A | — | — |
| FP-04 | No false positive on `clusterRef` and `clusterSelector` (NodePool) | GKE NodePool MR has `clusterRef` and `clusterSelector` in `forProvider`; GCP never reflects these Crossplane-only fields in `atProvider`. Run watcher. Confirm no drift on `clusterRef` or `clusterSelector`. | — | — | — | — |
| FP-05 | No false positive on absent map field group (nodeConfig on GKE Cluster) | GKE Cluster has `nodeConfig` in `forProvider`; after `removeDefaultNodePool`, GCP stops returning `nodeConfig` in `atProvider`. Run watcher. Confirm no drift reported on `nodeConfig`. | — | — | — | — |
| FP-06 | No false positive on computed-only `atProvider` fields (id, selfLink, etc.) | Run watcher against a live MR. Confirm fields like `id` and `selfLink` that exist only in `atProvider` do not appear in any drift report (diff iterates `forProvider` keys only). | — | — | — | — |
| FP-07 | No drift reported for resource without drift-protection label | Create a second MR of the same type without the `platform.io/drift-protection: "true"` label or with `platform.io/drift-protection: "false"` label. Make a change to it in GCP Console. Confirm the watcher does not report drift for it. | ✅ | — | — | — |
| FP-08 | No drift reported before atProvider is populated | Apply a new MR with drift protection. Before the provider runs `Observe()`, `atProvider` is empty. Run watcher. Confirm no drift is reported. | — | — | — | — |

---

## Section 4 — Deduplication

| ID | Summary | How To | A | B | C | D |
|----|---------|--------|:-:|:-:|:-:|:-:|
| DUP-01 | Rapid double-trigger fires exactly one workflow | Change a field in GCP Console while the watcher's first detection is in progress (or trigger two poll cycles in quick succession without resolving drift). Run: `temporal workflow list --query 'WorkflowType="DriftApprovalWorkflow" AND ExecutionStatus="Running"'`. Expected: exactly 1 result. | — | — | — | — |
| DUP-02 | Multi-MR XR fires exactly one workflow (GKE Cluster + NodePool) | Drift both the GKE Cluster MR and the NodePool MR at the same time (change cluster autoscaling profile AND node pool autoRepair). Both MRs resolve to the same XR via `ownerReferences`. Expected: only one `DriftApprovalWorkflow` running, keyed on the XR name. | — | — | — | — |
| DUP-03 | Watcher ignores `AlreadyStarted` from Temporal | While a `DriftApprovalWorkflow` is running (waiting for approval), trigger another poll cycle with the same drifted resource. Expected: no error logged, no second workflow started. Verify in watcher logs that `AlreadyStarted` is silently swallowed. | — | — | — | — |

---

## Section 5 — Observe Mode Behavior

| ID | Summary | How To | Result |
|----|---------|--------|:-:|
| OB-01 | No auto-heal while resource is in Observe mode | After drift is detected (e.g. tier changed in GCP Console), wait 5 min. Confirm the GCP resource still shows the drifted value and has not been corrected by Crossplane. `kubectl get <mr> <name> -o jsonpath='{.spec.managementPolicies}'` must still show `["Observe"]`. | ✅ |
| OB-02 | Crossplane still updates `atProvider` in Observe mode | While MR is in `managementPolicies: ["Observe"]`, GCP-side Observe() continues to run. Change a field. Verify `status.atProvider` is updated within 2 min even though Crossplane does not correct the resource. | ✅ |
| OB-03 | Crossplane should resume auto-heal resource when `["Create"]` or `["Update"]` is added to `managementPolicies` | While MR is in `managementPolicies: ["Observe"]`, and resource is modified or deleted (in GCP), add `["Create"]` or `["Update"]` into `managementPolicies`. | ✅ |

---

## Section 6 — Approval Workflow Paths

These cases require `DriftApprovalWorkflow` to be implemented.

| ID | Summary | How To | A | B | C | D |
|----|---------|--------|:-:|:-:|:-:|:-:|
| WF-01 | `DriftApprovalWorkflow` starts and enters `WAITING_FOR_APPROVAL` | After drift is detected, check Temporal UI. Workflow should be Running with status query returning `waiting_for_approval`: `temporal workflow query --workflow-id drift-approval-<ns>-<kind>-<xrName> --query-type approval-status` | — | — | — | — |
| WF-02 | `NotifyDriftActivity` fires with correct event=DRIFT_DETECTED | Check Temporal worker logs or Temporal UI event history for the `NotifyDriftActivity` call. Confirm `event=DRIFT_DETECTED`, correct `xrName`, `namespace`, and `driftDetail`. | — | — | — | — |
| WF-03 | Rejection path — workflow terminates, MR stays in Observe | Signal: `temporal workflow signal --workflow-id drift-approval-<...> --name approval-signal --input '{"approved":false}'`. Expected: workflow completes with `APPROVAL_REJECTED` error, `managementPolicies` remains `["Observe"]`, resource stays drifted in GCP. | — | — | — | — |
| WF-04 | Approval path — Crossplane reconciles and Observe is restored | Signal: `--input '{"approved":true}'`. Expected: R1 flips policy to `["*"]`, Crossplane reconciles (resource corrected in GCP), R2 waits for XR Ready=True, R3 flips policy back to `["Observe"]`. Verify final state: `managementPolicies=["Observe"]` and resource matches desired state. | — | — | — | — |
| WF-05 | R3 always runs even if reconciliation fails/times out | Approve the workflow on a resource that cannot reconcile (e.g. invalid tier). Expected: R2 times out or errors, but R3 still flips `managementPolicies` back to `["Observe"]`. MR is never left in full management mode. | — | — | — | — |
| WF-06 | Timeout path — 24h timeout fires `APPROVAL_TIMEOUT` notification | In a test environment, shorten the workflow timeout to 1 min. Wait for timeout. Confirm `NotifyDriftActivity` fires with `event=APPROVAL_TIMEOUT`, workflow exits with `APPROVAL_TIMEOUT` error, MR stays in Observe. | — | — | — | — |
| WF-07 | Watcher re-detects drift after rejection/timeout | After workflow exits (reject or timeout), the drift is still present. On the next poll cycle, the watcher should detect the same drift again and start a new workflow. Confirm a new `DriftApprovalWorkflow` with a new execution ID appears. | — | — | — | — |

---

## Section 7 — Approach-Specific Behavior

### Approach A — Polling Watcher

| ID | Summary | How To | Result |
|----|---------|--------|:-:|
| A-01 | Poll interval is respected | Set `DRIFT_POLL_INTERVAL=10s` in `.env`. Run watcher with `DRIFT_VERBOSE=true`. Observe `LIST_REQUEST` log entries. Verify they appear every ~10s, not faster. | ✅ |
| A-02 | `.env` loaded at runtime — change is reflected on restart | Stop watcher. Change `MR_GVRS` in `.env` (add or remove a GVR). Restart. Verify new GVR list is picked up without recompiling. | ✅ |
| A-03 | Environment variable overrides `.env` | Export `DRIFT_POLL_INTERVAL=60s` in shell. Start watcher (`.env` has `DRIFT_POLL_INTERVAL=30s`). Verify 60s interval is used (shell var wins). | — |
| A-04 | Restart after crash — already-drifted resource is re-detected | Kill the watcher while a drift event is in flight (before `ExecuteWorkflow` succeeds). Restart. Verify on the next poll the drifted resource is detected again. | — |

### Approach B — WatchOperations (Crossplane function-python)

| ID | Summary | How To | Result |
|----|---------|--------|--------|
| B-01 | WatchOperation triggers function on MR update | Update a watched MR (e.g. annotate it). Check `kubectl get operations -A` for a new Operation object. | — |
| B-02 | Drift detected in function-python logs | Check logs of the Crossplane function runner pod: `kubectl logs -n crossplane-system -l app=function-drift-notifier -f`. Confirm `DRIFT DETECTED` block appears. | — |
| B-03 | Function does NOT fire for unwatched resource | Create an MR without the drift-protection label. Confirm no WatchOperation watches it and no function invocation occurs. | — |

### Approach C — Informer Watcher

| ID | Summary | How To | Result |
|----|---------|--------|--------|
| C-01 | `UpdateFunc` fires within 1s of atProvider change | After triggering a field change in GCP (provider updates `atProvider` via PATCH), check watcher logs for `INFORMER_UPDATE` entry with `DRIFT_VERBOSE=true`. Confirm it appears within ~1s of the PATCH landing. | — |
| C-02 | `AddFunc` catches already-drifted resource on pod restart | Drift a resource, then kill the watcher pod before it detects the drift. Restart pod. The informer's `AddFunc` fires for each existing watched MR on cache sync. Confirm `DRIFT DETECTED` appears in logs for the already-drifted resource. | — |
| C-03 | Resync (10 min) catches events missed during failure window | Stop the watcher for 10+ min while drift exists. Restart. Verify the 10-min resync fires `AddFunc` for the drifted resource even if the update event was missed. | — |
| C-04 | Adding a new GVR requires pod restart | Add a new GVR to `.env`. Restart pod. Verify `WATCH_SETUP` log entry for the new GVR appears on startup. | — |

### Approach D — Temporal Schedule

| ID | Summary | How To | Result |
|----|---------|--------|--------|
| D-S01 | Scan schedule fires every minute | `temporal schedule describe --schedule-id drift-scan`. Confirm `RecentActions` shows runs at 1-min intervals. | — |
| D-S02 | `DriftScanWorkflow` output reports correct counts | After a scan with 3 resources watched and 1 drifted, check Temporal UI for the completed `DriftScanWorkflow`. Result should show `scannedResources: 3, driftedResources: 1`. | — |
| D-S03 | Scan continues if one GVR errors (list error tolerance) | Make one GVR in the schedule config invalid (typo in group name). Confirm the scan still runs and reports results for valid GVRs. The invalid GVR logs an error but does not abort the activity. | — |
| D-S04 | `DRIFT_VERBOSE=true` shows list requests in activity logs | Set `DRIFT_VERBOSE=true` in worker `.env`. Check Temporal worker pod logs during a scan. Confirm `LIST_REQUEST` and `LIST_RESPONSE` entries appear per GVR. | — |

---

## Section 8 — Edge Cases

| ID | Summary | How To | A | B | C | D | Notes |
|----|---------|--------|:-:|:-:|:-:|:-:|---------|
| EC-01 | `atProvider` is empty (provider hasn't observed yet) | Apply a new MR with drift protection but immediately pause `provider-gcp` before it runs `Observe()`. Check `atProvider` is empty. Run watcher. Expected: no drift reported (`atProvider` empty check gates Signal 1). | — | — | — | — | Tested by hardcoding the empty `atProvider` |
| EC-02 | `forProvider` is empty (edge case resource) | If a composition produces an MR with an empty `forProvider` (unlikely but defensive). Expected: no diff reported (nothing to compare against). | — | — | — | — | Tested by hardcoding the empty `forProvider` |
| EC-03 | Resource has drift-protection label but `driftProtection: false` in XR spec | MR is being watched (has label) but is in full management mode (`managementPolicies: ["*"]`). Drift is introduced. Expected: drift is still detected (label-based watch is independent of policy). Note: Crossplane may also auto-heal in this case — verify behavior. | ✅ | — | — | — |
| EC-04 | Temporal unavailable when watcher tries to fire workflow | Stop Temporal or block network to it while drift is detected. Expected: **A/C** — error logged, next poll retries. **D** — next schedule run retries. Verify no silent data loss. | — | — | — | — |
| EC-05 | Large number of drifted resources (fan-out) | Have 5+ resources all drifted simultaneously. Expected: one `DriftApprovalWorkflow` fires per XR (not per MR). Multi-MR XRs fire only once. Verify workflow count = number of distinct XRs, not MRs. | — | — | — | — |
| EC-06 | Field drift and deletion occur simultaneously (both signals fire for same MR) | Externally modify a field AND delete the resource in quick succession. Both Signal 1 and Signal 2 should appear in the same drift report (`changes (2)`). | — | — | — | — |
| EC-07 | Observed value is `nil` vs desired is a non-empty map | A GCP resource configuration removes a field group from `atProvider` that was present in `forProvider` (e.g. optional feature disabled, GCP stops returning the section). Expected: treated as no-diff (no observed baseline = cannot conclude drift). | — | — | — | — |
| EC-08 | Desired slice has elements but observed slice is shorter | `forProvider` declares 3 node locations, GCP only reports 2 in `atProvider`. Expected: the missing index is reported as `<missing>` in the drift output. | — | — | — | — |

---

## Checklist Before Marking POC Complete

- [ ] All S-0x setup checks pass on all four branches
- [ ] D-01 through D-09 pass on all four branches
- [ ] FP-01 through FP-08 pass on all four branches (no false positives)
- [ ] DUP-01 through DUP-03 pass on all four branches
- [ ] OB-01 and OB-02 pass on all four branches
- [ ] WF-01 through WF-07 pass (once `DriftApprovalWorkflow` is implemented)
- [ ] Approach-specific cases (A-01–A-04, B-01–B-03, C-01–C-04, D-S01–D-S04) pass per branch
- [ ] EC-01 through EC-08 verified
- [ ] Latency tables in `comparison-metrics.md` filled in (D-01/D-07 runs, 3 times each)
