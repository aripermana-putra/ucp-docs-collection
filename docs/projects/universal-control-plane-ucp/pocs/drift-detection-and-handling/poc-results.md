---
title: "POC Results"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Drift Detection POC — Results

This document is the primary output of the drift detection POC. It covers approach
benchmarking, key findings discovered during execution, and the recommended production
architecture.

---

## Branches

| Approach | Branch |
|----------|--------|
| A — Polling Watcher | `feature/drift-poc-approach-a-watcher` |
| B — WatchOperations | `feature/drift-poc-approach-b-watchoperations` |
| C — Informer Watcher | `feature/drift-poc-approach-c-informer` |
| D — Temporal Schedule | `feature/drift-poc-approach-d-temporal-schedule` |

---

## Metric 1: Drift Detection Latency

**Definition:** Time from GCP-side change to `DriftApprovalWorkflow` appearing as Running
in Temporal UI.

**Two drift scenarios require two different detection signals:**

| Drift type | GCP-side event | Detection signal | Why the other signal fails |
|---|---|---|---|
| Field change | Field modified (e.g. tier, labels) in GCP console | `forProvider vs atProvider` diff, after Observe() updates `atProvider` | `Synced` may remain `True` — diff is the only signal |
| Resource deletion | Resource deleted from GCP console | `.status.conditions[type=Synced, status=False, reason=ReconcileError]` | `atProvider` is cleared or stale after deletion — diff alone **misses this** |

Both signals must be implemented in each approach. Neither alone is sufficient.

**Note:** All approaches share a ~1 min floor because the provider `--poll` interval is the
bottleneck — the watcher can only react once `atProvider` is updated or `Synced` flips.
Configured to `--poll=1m` (default is 10m) and `--sync=30m` (default is 1h) via
`DeploymentRuntimeConfig` in `crossplane/providers/gcp/provider-gcp-sql.yaml`.
See crossplane-reconcile-behavior.md.

| Approach | Expected gap after signal | Total expected latency |
|----------|--------------------------|-----------------------|
| A | 0–30s (poll interval) | 60–90s |
| B | <1s (event-driven) | ~60s |
| C | <1s (event-driven) | ~60s |
| D | 0–60s (schedule interval) | 60–120s |

**Results (fill in after POC — run E2E test 3 times per approach, for each drift type):**

*Field drift (field modified in GCP console):*

| Approach | Run 1 | Run 2 | Run 3 | Average |
|----------|-------|-------|-------|---------|
| A | | | | |
| B | | | | |
| C | | | | |
| D | | | | |

*Resource deletion (deleted from GCP console):*

| Approach | Run 1 | Run 2 | Run 3 | Average |
|----------|-------|-------|-------|---------|
| A | | | | |
| B | | | | |
| C | | | | |
| D | | | | |

---

## Metric 2: Setup Complexity

**Definition:** Number of distinct setup steps before the first successful E2E run.

| Requirement | A | B | C | D |
|-------------|---|---|---|---|
| Alpha Crossplane flags | No | Yes (`--enable-operations`) | No | No |
| New K8s Deployments | 1 | 0 | 1 | 0 |
| Container image to build+push | No | Yes (function-python) | No | No |
| New languages/runtimes | No | Yes (Python) | No | No |
| New Crossplane CRD types | No | Yes (WatchOperation, Operation) | No | No |
| One-time setup commands | ConfigMap apply | Crossplane reinstall + image push | ConfigMap apply | `temporal schedule create` |

**Complexity score** (count "Yes" entries): A=1, B=5, C=1, D=1

---

## Metric 3: Code Volume

**Measured from actual source files (detection logic only; XRD/composition changes shared across all approaches not included):**

| Approach | Go LOC | Python LOC | YAML LOC | Notes |
|----------|-------:|----------:|--------:|-------|
| A | 298 | — | ~80 | `drift-watcher/main.go` + k8s deployment/configmap (estimated, not yet created) |
| B | — | 182 | 149 | `drift-notifier/main.py` + 6 WatchOperation YAMLs (156) + Dockerfile (11) |
| C | 311 | — | ~80 | `drift-watcher/main.go` + k8s deployment/configmap (estimated, not yet created) |
| D | 292 | — | — | `drift_scan.go` activity (259) + `drift_scan.go` workflow (33) + setup script (27 shell, not YAML) |

> **Note:** Shared files not yet implemented: `drift_approval.go` (workflow) and `drift.go` (FlipManagementPolicyActivity, NotifyDriftActivity) will add roughly equal LOC to all approaches.

---

## Metric 4: Multi-Resource Extensibility

**When a new resource type is added, what changes?**

| Change | A | B | C | D |
|--------|---|---|---|---|
| Code change | No | No | No | No |
| New manifest/config | ConfigMap line | New WatchOperation YAML | ConfigMap line | Schedule update cmd |
| Restart required | No | No | Yes (pod) | No |
| XRD change | `driftProtection` field | same | same | same |
| Composition change | conditional policies | same | same | same |

All approaches require XRD + composition changes per new resource type (inherent to
ManagementPolicy mechanism). The trigger extension overhead is what differs.

---

## Metric 5: Debuggability

**When drift is detected but no workflow fires, how quickly can you diagnose it?**

| Scenario | A | B | C | D |
|----------|---|---|---|---|
| Drift not detected | Pod logs + XR conditions — cannot tell if watcher missed it or provider never observed | `kubectl describe watchoperation` + `kubectl get operations` | Informer heartbeat distinguishes two failure modes: no `INFORMER_UPDATE` = provider not observing; update fired but no drift log = watcher ran clean | Temporal scan workflow result |
| Workflow fires but stalls | Temporal UI | Temporal UI | Temporal UI | Temporal UI |
| ManagementPolicy not flipping | `kubectl get xr -o yaml` + Temporal events | Same | Same | Same |

**Score:** 1=easiest (<5 commands), 3=hardest (>10 commands)

| Approach | Score | Notes |
|----------|:-----:|-------|
| A | 2 | Pod logs + `kubectl get xr -o yaml`; cannot distinguish provider-silent from watcher-missed |
| B | 2 | `kubectl describe watchoperation` + `kubectl get operations`; structured but more objects to inspect |
| C | 1 | Informer events act as a provider heartbeat — absence of `INFORMER_UPDATE` pinpoints the problem upstream at the provider, not the watcher; no ambiguity |
| D | 1 | Temporal UI shows every scan run with structured output (scanned/drifted counts); fastest to diagnose |

---

## Metric 6: Observability Out-of-the-Box

| Observable surface | A | B | C | D |
|-------------------|---|---|---|---|
| Currently monitored resources | `kubectl get xr -l platform.io/drift-protection=true` | Same | Same | Same |
| Active drift events | Temporal UI (running DriftApprovalWorkflows) | Temporal UI + `kubectl get operations` | Temporal UI | Temporal UI |
| Detection history | Pod logs | `kubectl get operations` + Temporal UI | Pod logs | Temporal UI scan history |
| Scan metrics (how many checked) | No | No | No | Yes (DriftScanOutput) |
| Trigger component health | `kubectl get pod drift-watcher` | `kubectl describe watchoperation` | `kubectl get pod drift-watcher` | Temporal Schedule status |

**Notable:** D has the richest built-in observability — every scan run is a workflow with
structured output (scanned/drifted/fired counts) visible in the Temporal UI.

---

## Metric 7: Failure Resilience

**What happens to a drift event if a component fails mid-detection?**

| Failure scenario | A | B | C | D |
|-----------------|---|---|---|---|
| Trigger crashes after drift, before workflow start | Event lost (poll window) | Operation persisted in etcd — Crossplane retries | **Event NOT lost** (AddFunc + resync) | Event not lost (next schedule run) |
| Temporal unavailable when firing | Next poll retries | Next Operation retries | Next event/resync retries | Next schedule run retries |

**Score:** 1=no data loss risk, 3=data loss possible

| Approach | Score | Notes |
|----------|:-----:|-------|
| A | 2 | Crash between drift detection and `ExecuteWorkflow` loses that event; next poll recovers if drift persists |
| B | 1 | Operation object persisted in etcd by Crossplane; survives function crashes, retried automatically |
| C | 1 | `AddFunc` fires for already-drifted MRs on restart; 10-min resync catches anything missed during failures |
| D | 2 | Next schedule run (≤1 min) picks up all drifted resources; no data loss if Temporal itself is healthy |

---

## Metric 8: Deduplication Correctness

**Test:** Trigger the same resource drift 4 times rapidly, verify exactly 1 running workflow.

```bash
temporal workflow list \
  --query 'WorkflowType="DriftApprovalWorkflow" AND ExecutionStatus="Running"'
# Expected: exactly 1 result
```

All approaches use Temporal workflow ID dedup — expected pass for all. Record any
unexpected behavior.

| Approach | Pass/Fail | Notes |
|----------|:---------:|-------|
| A | Expected Pass | Temporal workflow ID dedup — pending E2E confirmation |
| B | Expected Pass | Temporal workflow ID dedup — pending E2E confirmation |
| C | Expected Pass | Temporal workflow ID dedup — pending E2E confirmation |
| D | Expected Pass | Temporal workflow ID dedup — pending E2E confirmation |

---

## Metric 9: K8s API Load at Scale

**Steady state with 50 resources across 4 GVRs, nothing drifted:**

| Approach | Call pattern | Rate |
|----------|-------------|------|
| A | LIST per GVR per poll | 4 × 2/min = 8 list calls/min |
| B | WATCH (persistent) | 4 watch streams, idle |
| C | WATCH (persistent) | 4 watch streams, idle |
| D | LIST per GVR per scan | 4 × 1/min = 4 list calls/min |

```bash
# Measure actual rate with 5 test resources:
kubectl get --raw /metrics | grep apiserver_request_total | grep list
```

| Approach | Measured rate | Score (1=lowest load) |
|----------|:------------:|:--------------------:|
| A | ~8 list calls/min (5 GVRs × 2/min) | 3 |
| B | 5 persistent watch streams, idle | 1 |
| C | 5 persistent watch streams, idle | 1 |
| D | ~5 list calls/min (5 GVRs × 1/min) | 2 |

> **Note:** Rates above are theoretical at steady state with 0 drift and 5 GVRs configured. Actual measurement pending.

---

## Metric 10: Production Readiness Path

| Gap | A | B | C | D |
|-----|---|---|---|---|
| Real notifications (Slack/PD) | Swap `NotifyDriftActivity` body | Same | Same | Same |
| High availability | Scale Deployment replicas | Crossplane HA | Scale replicas | Temporal worker HA |
| Alpha → stable gap | None | WatchOperations must graduate | None | None |
| Audit trail | Add structured logging | Operation history + Temporal | Logging | Full Temporal scan history |
| Go from POC to prod | Minimal changes | Requires alpha graduation | Minimal | Minimal |

---

## Decision Scorecard

**Scoring:** 4=best for this metric, 1=worst. Ties allowed.

**Scoring:** 4=best for this metric, 1=worst. Based on design analysis — update after E2E test results.

| Metric | A | B | C | D | Rationale |
|--------|:-:|:-:|:-:|:-:|-----------|
| Detection latency | 3 | 4 | 4 | 2 | B/C event-driven (~60s); A adds 0–30s poll lag; D adds 0–60s schedule lag |
| Setup complexity | 4 | 1 | 4 | 4 | B requires alpha flag + Crossplane reinstall + image build + push |
| Code volume | 3 | 3 | 2 | 4 | D least new code (no binary); C slightly more than A (informer setup); B has Python overhead |
| Multi-resource extensibility | 3 | 2 | 2 | 4 | D: schedule update only; A: ConfigMap line; C: ConfigMap + pod restart; B: new YAML file |
| Debuggability | 2 | 2 | 4 | 4 | C: informer heartbeat pinpoints provider-silent vs watcher-missed with no ambiguity; D: Temporal UI scan history; A/B: logs only, cannot distinguish failure modes |
| Observability | 2 | 3 | 2 | 4 | D: structured scan output per run in Temporal UI; B: Operations visible in cluster; A/C: logs only |
| Failure resilience | 2 | 3 | 4 | 2 | C: AddFunc+resync never loses events; B: Operation in etcd survives crash; A/D: small loss window |
| Deduplication correctness | 4 | 4 | 4 | 4 | All use Temporal workflow ID dedup — expected pass |
| K8s API load | 3 | 1 | 1 | 2 | B/C: idle watch streams (C bounded by provider rate, not its own schedule); A: independent poll schedule, 8 list/min at 10s interval; D: 5 list/min |
| Production readiness | 4 | 1 | 4 | 4 | B: blocked on WatchOperations alpha graduation; others production-ready today |
| **Total** | **30** | **24** | **31** | **34** | |

---

## E2E Test Script

Run identically on all four branches:

```bash
#!/bin/bash
# Usage: ./drift-e2e-test.sh [xr-name] [namespace]
set -e
XNAME="${1:-drift-poc-test}"
NAMESPACE="${2:-default}"

echo "=== [1] Apply XDatabase with driftProtection + label ==="
kubectl apply -f crossplane/examples/dbaas/cloudsql/xdatabase-postgres.yaml
kubectl wait xdatabase/$XNAME -n $NAMESPACE --for=condition=Ready --timeout=30m

echo "=== [2] Confirm Observe mode ==="
kubectl get databaseinstance -A -o jsonpath='{.items[0].spec.managementPolicies}'
# Expected: ["Observe"]

echo "=== [3] Simulate drift — delete from GCP Console, then press Enter ==="
read -p "Press Enter after GCP deletion..."
T0=$(date +%s)

echo "=== [4] Wait for Synced=False ==="
until kubectl get xdatabase $XNAME -n $NAMESPACE \
  -o jsonpath='{.status.conditions}' 2>/dev/null | grep -q '"reason":"ReconcileError"'; do
  sleep 5
done
T_SYNCED=$(date +%s)
echo "Synced=False at +$((T_SYNCED - T0))s"

echo "=== [5] Wait for DriftApprovalWorkflow to start ==="
until temporal workflow list \
  --query 'WorkflowType="DriftApprovalWorkflow" AND ExecutionStatus="Running"' \
  2>/dev/null | grep -q "drift-approval"; do
  sleep 2
done
T1=$(date +%s)
echo ">>> Total detection latency: $((T1 - T0))s"
echo ">>> Gap after Synced=False:  $((T1 - T_SYNCED))s"

echo "=== [6] Verify no auto-heal after 2 minutes ==="
sleep 120
POLICIES=$(kubectl get databaseinstance -A -o jsonpath='{.items[0].spec.managementPolicies}')
echo "$POLICIES" | grep -q "Observe" && echo "PASS: still in Observe" || echo "FAIL: $POLICIES"

echo "=== [7] Test rejection ==="
WF_ID="drift-approval-${NAMESPACE}-xdatabase-${XNAME}"
temporal workflow signal --workflow-id "$WF_ID" --name "approval-signal" \
  --input '{"approved": false}'
sleep 5
echo "Workflow terminated on rejection."

echo "=== [8] Trigger drift again for approval path ==="
read -p "Delete from GCP Console again, then press Enter..."
T0=$(date +%s)
until temporal workflow list \
  --query 'WorkflowType="DriftApprovalWorkflow" AND ExecutionStatus="Running"' \
  2>/dev/null | grep -q "drift-approval"; do sleep 2; done
T1=$(date +%s)
echo "New workflow started (+$((T1 - T0))s)"

echo "=== [9] Approve ==="
temporal workflow signal --workflow-id "$WF_ID" --name "approval-signal" \
  --input '{"approved": true}'

echo "=== [10] Wait for reconciliation + Observe restore ==="
kubectl wait xdatabase/$XNAME -n $NAMESPACE --for=condition=Ready --timeout=35m
sleep 15
POLICIES=$(kubectl get databaseinstance -A -o jsonpath='{.items[0].spec.managementPolicies}')
echo "$POLICIES" | grep -q "Observe" && echo "PASS: Observe restored" || echo "FAIL: $POLICIES"

echo "=== E2E complete. Record results in Comparison Metrics. ==="
```

---

## Findings

Issues and behaviors discovered during POC execution that affect the design.

---

### F-01: Auto-reconciliation is unreliable for deletion drift on complex resources

**Observed during:** GKE Cluster + NodePool deletion recovery test (Approach A).

When a resource is deleted from GCP and Crossplane attempts to recreate it (after switching
from `Observe` to `Create` management policy), the recreate fails due to stale fields in
`forProvider` left behind by late initialization.

**What is late initialization:** After Crossplane creates a resource, the provider runs
`Observe()` and writes GCP's full API response back into `forProvider` — including fields the
composition never declared. These are cluster- or resource-specific values GCP assigned
automatically (pod secondary range names, logging component lists, etc.).

**Failure sequence encountered:**

1. GKE Cluster deleted from GCP Console.
2. Management policy switched to `["Observe", "Create"]`.
3. Cluster recreate attempted — **failed**: `Error 400: Cannot specify logging_config
   together with logging_service`. The composition had `loggingService: "none"` (old
   deprecated field); late initialization had also written `loggingConfig` (new field) into
   `forProvider` from the original cluster's observed state. GCP rejects requests with both
   field generations simultaneously.
4. Composition updated to use only `loggingConfig`/`monitoringConfig` (new style). Cluster
   recreate retried — **failed again**: `Error 400: SYSTEM_COMPONENTS monitoring must be
   enabled if any monitoring is enabled`. The late-initialized `forProvider.monitoringConfig`
   from the old cluster had a component list missing `SYSTEM_COMPONENTS`.
5. Composition fixed with `enableComponents: []`. Cluster created. NodePool recreate
   attempted — **failed**: pod secondary range
   `gke-ari-test-kube-cluster-1-pods-0f92ba7c` not found. The late-initialized
   `forProvider.networkConfig.podRange` referenced a range that belonged to the deleted
   cluster and no longer existed.

**Root cause summary:** Late initialization is additive and cumulative. Each `Observe()`
cycle writes more GCP-managed fields into `forProvider`. When the resource is deleted and
recreated, those fields reference state that belonged to the old resource and is now gone.

**Scope of impact:**

| Drift type | Auto-reconciliation | Why |
|---|---|---|
| Field change (resource still exists) | ✅ Works | Crossplane sends UPDATE to existing resource; late-init values are still valid |
| Resource deletion | ❌ Unreliable | Crossplane sends CREATE; stale late-init fields reference deleted resource state |

**Recommendation for approval workflow:** When the drift signal is `Synced=False/ReconcileError`
(deletion), the approval flow should delete the composed MRs before flipping management
policy. This forces the composition to recreate them from scratch with a clean `forProvider`.
For field-level drift, no change needed — the existing flow works.

**Discussion point:** Whether Crossplane should clear late-initialized fields from `forProvider`
when a resource is confirmed deleted (Synced=False/ReconcileError), rather than leaving stale
values that make recreation fail.

---

### F-02: Approach C's informer events are coupled to the provider poll cycle — enabling provider heartbeat detection

**Observed during:** Approach C informer watcher steady-state run.

When the Crossplane provider runs `Observe()`, it writes the result back to `status.atProvider`
on the MR even if nothing in GCP changed. This increments `resourceVersion`, which triggers a
`MODIFIED` event on the Kubernetes watch stream. The informer fires `UpdateFunc` on every one
of these writes.

**Implication for K8s API load (corrects Metric 9):** Approach A makes its own LIST calls on
its own schedule regardless of provider activity. At `DRIFT_POLL_INTERVAL=10s`, that is 6 LIST
calls per minute per GVR whether the provider observed anything or not. Approach C makes zero
calls of its own — it only wakes up when the provider writes. C's work is strictly bounded by
the provider poll rate; A's is not. The "event-driven" label on C is accurate not just for
drift reaction time but also for steady-state load.

**Provider heartbeat as a free side-effect:** Because every `UpdateFunc` event in C corresponds
directly to a provider `Observe()` cycle, the timestamp of each event is "last time this
resource was observed by the provider". This is available for free — no extra API calls needed.

Useful signal this enables:

- Per-resource "last observed at" metric surfaced from informer event timestamps
- Health check: if `now - lastObservedAt > 2 × pollInterval`, the provider may be stuck or the
  MR is no longer being reconciled
- Provider liveness detection without parsing provider logs

**No other approach provides this signal without extra work:**

| Approach | Provider observation signal | Notes |
|---|---|---|
| A | ❌ | Lists on its own schedule, not correlated to provider activity |
| B | ✅ Partial | Function invoked on MR changes, but no persistent storage across invocations |
| C | ✅ Native | Each `UpdateFunc` = one provider `Observe()` cycle |
| D | ❌ | Scans on Temporal schedule, decoupled from provider activity |

---

## Recommended Production Architecture

Based on expected POC results, the likely outcome:

- **C wins overall** — event-driven, Go, no alpha deps, resync safety net, lowest K8s API load
- **D is a strong second** — richest observability, fewest moving parts, but Temporal-availability-coupled
- **A proves the concept fastest** — implement A first, migrate to C for production
- **B is the future** — most architecturally elegant once WatchOperations graduates from alpha

**Recommended hybrid** if C and D are close:

> Use Approach C for detection (informer-based, event-driven) and leverage Temporal's
> workflow history (from D) for observability by ensuring `DriftScanWorkflow`-equivalent
> metrics are emitted as Temporal workflow signals or search attributes.
