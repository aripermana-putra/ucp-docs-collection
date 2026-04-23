---
title: "POC Results"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Drift Detection POC — Results

This document is the primary output of the drift detection POC. It covers approach
benchmarking, key findings discovered during execution, and the POC summary and decision.

---

## Branches

| Approach | Branch |
|----------|--------|
| A — Polling Watcher | `feature/MCUUCP-125-drift-poc-approach-a-watcher` |
| B — WatchOperations | `feature/MCUUCP-125-drift-poc-approach-b-watchoperations` |
| C — Informer Watcher | `feature/MCUUCP-125-drift-poc-approach-c-informer` |
| D — Temporal Schedule | `feature/MCUUCP-125-drift-poc-approach-d-temporal-schedule` |

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
See [crossplane-reconcile-behavior](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6595619547).


| Approach | Expected gap after signal | Total expected latency |
|----------|--------------------------|-----------------------|
| A | 0–30s (poll interval) | 60–90s |
| B | <1s (event-driven) | ~60s |
| C | <1s (event-driven) | ~60s |
| D | 0–30s (schedule interval) | 60–90s |

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
| Drift not detected | Pod logs + XR conditions — cannot tell if watcher missed it or provider never observed | `kubectl describe watchoperation` + `kubectl get operations` | Informer heartbeat distinguishes two failure modes for GKE (container provider writes unconditionally): no `INFORMER_UPDATE` = provider not observing; update fired but no drift log = watcher ran clean. For CloudSQL/Compute/Storage: silent in steady state — same ambiguity as A | Temporal scan workflow result |
| Workflow fires but stalls | Temporal UI | Temporal UI | Temporal UI | Temporal UI |
| ManagementPolicy not flipping | `kubectl get xr -o yaml` + Temporal events | Same | Same | Same |

**Score:** 1=easiest (<5 commands), 3=hardest (>10 commands)

| Approach | Score | Notes |
|----------|:-----:|-------|
| A | 2 | Pod logs + `kubectl get xr -o yaml`; cannot distinguish provider-silent from watcher-missed |
| B | 2 | `kubectl describe watchoperation` + `kubectl get operations`; structured but more objects to inspect |
| C | 2 | Informer heartbeat works for GKE (container provider writes unconditionally every cycle); silent for CloudSQL, Compute, Storage in steady state — heartbeat benefit is `provider-upbound-gcp-container`-specific, not general |
| D | 2 | Temporal UI shows scan results (scanned/drifted counts) but cannot tell if the provider ran Observe(); "drifted: 0" is ambiguous — provider may not have updated atProvider |

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

**Watch stream cost model for B and C:**

Watch streams use HTTP/2 — all GVR streams are multiplexed over a single TCP connection to
the API server, so 5 GVR watches = 1 TCP connection with 5 HTTP/2 streams. The client-side
cost is minimal. However, the cost model has a server side that polling approaches do not:

| Cost | Polling (A, D) | Watch stream (B, C) |
|------|---------------|---------------------|
| Network | Periodic request/response | TCP keepalives when idle; event push on change |
| API server | Processes each LIST independently | Maintains watcher state + fans out every event to all registered watchers |
| Failure recovery | Next scheduled poll catches up cleanly | Reconnect triggers a full LIST + re-watch burst per GVR |
| Predictability | Uniform, scheduled load | Mostly idle, but bursty on pod restart or network blip |
| **Watcher memory** | **Periodic spike during poll → GC → near zero** | **Linear steady-state — every labeled MR cached in heap permanently** |

The reconnect LIST burst is the main API server concern. However, for large-scale platforms
the **memory footprint of the local cache is the harder constraint** — see below.

**⚠️ In-memory cache scaling concern (Approach C specific):**

The SharedInformer caches every watched object in the watcher pod's heap. This is fundamental
to the informer model and cannot be opted out of. Approaches A and D do not cache anything.

| Labeled MR count | Avg object size | Approach C heap | Approach A/D heap |
|-----------------|----------------|-----------------|------------------|
| 1,000 | 50 KB | ~50 MB | ~flat |
| 10,000 | 50 KB | ~500 MB | ~flat |
| 50,000 | 50 KB | ~2.5 GB | ~flat |

At platform scale serving a large organisation with drift protection enabled broadly, this
becomes a hard memory ceiling per watcher pod. Mitigation options: label discipline (opt-in
only for critical resources), sharding watchers by namespace/provider, or switching to
Approach A/D at that scale.

Confirmed from client-go v0.30.0 source — every object is stored in
`threadSafeMap.items map[string]interface{}` (`tools/cache/thread_safe_store.go:379`).
Resync is also confirmed local-only: `threadSafeMap.Resync()` is literally `// Nothing to do`
(`tools/cache/thread_safe_store.go:371`). No API calls in the resync path.

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

## Metric 11: Scalability

**Definition:** How does each approach behave as the number of labeled MRs and GVRs grows?
Covers memory model, scale-out mechanism, and per-component peak load.

**Memory models differ fundamentally across approaches:**

| Approach | Memory pattern | Peak per poll/event | Average over time |
|----------|---------------|--------------------|--------------------|
| A | Periodic spike during poll → GC → near zero | `GVRs × MRs_per_GVR × obj_size` | Low (near-zero between polls) |
| B | No persistent process — Operation pod exits after each event | ~obj_size per event | Near-zero (ephemeral pods) |
| C | Constant steady-state — every labeled MR cached in heap forever | `total_labeled_MRs × obj_size` | High (never GC'd) |
| D | Periodic spike distributed across Temporal worker fleet | `~obj_size × MRs_per_GVR` per worker | Low (GC'd after each activity) |

```
At 20 GVRs × 1,000 MRs × 50 KB:

Approach A:  1 pod, ~1 GB spike per poll cycle → GC
Approach B:  no persistent state; ~50 KB per Operation pod, exits after event
Approach C:  1 pod, ~1 GB permanent heap (never collected)
Approach D:  ~50 MB per worker pod per GVR activity, distributed across fleet → GC
```

**Scale-out mechanism:**

| Approach | How to scale out | Manual coordination required |
|----------|-----------------|------------------------------|
| A | Split GVRs across multiple watcher pods via separate ConfigMaps, or shard by namespace | Yes — you manage the split manually |
| B | Crossplane manages Operation pod scheduling; increase `spec.concurrencyPolicy` if needed | No — Crossplane schedules pods |
| C | Shard watchers by namespace/provider/GVR group across pods | Yes — you manage the split manually |
| D | Add more Temporal worker pods; Temporal distributes activities automatically | No — Temporal routes to available workers |

**Scale projection at 20 GVRs × 1,000 MRs (realistic platform scale):**

| Approach | Pod count | Per-pod memory | Total cluster memory | Scale-out path |
|----------|:---------:|:--------------:|:--------------------:|----------------|
| A | 1 | ~1 GB (spike) | ~1 GB | Manual sharding |
| B | 1 per event (ephemeral) | ~50 KB | Negligible | Automatic (Crossplane concurrency) |
| C | 1 | ~1 GB (steady) | ~1 GB | Manual sharding |
| D | N workers (existing fleet) | ~50 MB (spike, distributed) | ~50 MB × GVRs / workers | Automatic (Temporal routing) |

**Score:** 4=best scalability, 1=worst

| Approach | Score | Notes |
|----------|:-----:|-------|
| A | 2 | Periodic spike avoids persistent cache; but no built-in scale-out — manual sharding required at high GVR×MR counts |
| B | 3 | No persistent memory accumulation; Operation pods are ephemeral and K8s-scheduled; scale limited by Crossplane concurrency policy and cluster pod capacity |
| C | 1 | Worst at scale — linear heap growth with no GC, single pod; sharding adds operational complexity without solving the fundamental cache model |
| D | 4 | Best natural scale-out — Temporal distributes activity execution across the worker fleet automatically; distributed spikes instead of concentrated ones; no manual pod-to-GVR mapping |

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
| Debuggability | 2 | 2 | 3 | 3 | C: informer heartbeat works for GKE (container provider) but not for CloudSQL/Compute/Storage — partial advantage; D: Temporal UI shows scan results but cannot tell if provider ran Observe(); A/B: logs only |
| Observability | 2 | 3 | 2 | 4 | D: structured scan output per run in Temporal UI; B: Operations visible in cluster; A/C: logs only |
| Failure resilience | 2 | 3 | 4 | 2 | C: AddFunc+resync never loses events; B: Operation in etcd survives crash; A/D: small loss window |
| Deduplication correctness | 4 | 4 | 4 | 4 | All use Temporal workflow ID dedup — expected pass |
| K8s API load | 3 | 1 | 2 | 2 | B/C: fewest API calls (idle watch streams); A: 8 list/min independent of provider. C scores 2 not 1: low API call volume but trades it for linear memory growth — every labeled MR cached in heap permanently. At 10,000+ labeled resources C's memory footprint becomes a harder constraint than A/D's periodic LIST calls. B shares the same watch model but is not a long-running binary so cache concern is lower. |
| Production readiness | 4 | 1 | 4 | 4 | B: blocked on WatchOperations alpha graduation; others production-ready today |
| Scalability | 2 | 3 | 1 | 4 | D: Temporal distributes activities across worker fleet automatically — best natural scale-out; B: ephemeral Operation pods, no persistent cache; A: periodic spikes but manual sharding only; C: linear persistent heap growth, single pod, hardest to scale |
| **Total** | **32** | **27** | **32** | **37** | |

---

## Test Cases

Check [here](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6597720300) for the test cases used for this PoC and results on each approach.

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

### F-02: Informer heartbeat is specific to `provider-upbound-gcp-container` — not a general Crossplane behavior

**Observed during:** Approach C informer watcher steady-state run. All four resource types
were deployed with `platform.io/drift-protection=true` and all providers configured at the
same `--poll=1m --sync=30m` interval.

**What was expected:** Every provider `Observe()` cycle would increment `resourceVersion` and
fire an `INFORMER_UPDATE` log for each resource type.

**What was actually observed across all four resource types:**

| Provider binary | Resource type | Steady-state informer events |
|---|---|---|
| `provider-upbound-gcp-container` | GKE Cluster | `INFORMER_UPDATE` every ~1 min ✅ |
| `provider-upbound-gcp-container` | GKE NodePool | `INFORMER_UPDATE` every ~1 min ✅ |
| `provider-upbound-gcp-sql` | CloudSQL DatabaseInstance | Silent ❌ |
| `provider-upbound-gcp-compute` | Compute Instance | Silent ❌ |
| `provider-upbound-gcp-storage` | Storage Bucket | Silent ❌ |

**Root cause — proven by watch stream inspection:**

A watch stream diff (`kubectl get --watch -o json` with `resourceVersion` stripped) confirmed
that GKE writes are **no-op writes** — the content is byte-for-byte identical on every cycle,
yet Kubernetes still increments `resourceVersion` and fires a `MODIFIED` watch event. The
content of `status.atProvider`, `status.conditions`, and `metadata` does not change.

This means `provider-upbound-gcp-container` calls `Status().Update()` **unconditionally** on
every reconcile cycle. Kubernetes accepts the write, writes it to etcd, and increments
`resourceVersion` — even when the payload is identical to what is already stored.

The three other provider binaries (`sql`, `compute`, `storage`) do **not** call
`Status().Update()` unless something actually changed — so in steady state they produce no
writes, no `resourceVersion` increments, and no informer events.

**This is a `provider-upbound-gcp-container`-specific implementation detail**, not a general
Crossplane or upjet behavior. Other providers in the same upjet family do not exhibit it.

**Implication for provider heartbeat:** The heartbeat benefit of approach C is real but applies
only to resources managed by `provider-upbound-gcp-container`. It does not generalize.

| Approach | Provider observation signal | Notes |
|---|---|---|
| A | ❌ | Lists on its own schedule, not correlated to provider activity |
| B | ✅ Partial | Function invoked on MR changes, but no persistent storage across invocations |
| C | ✅ Container provider only | Heartbeat works for GKE Cluster/NodePool; silent for CloudSQL, Compute, Storage in steady state |
| D | ❌ | Scans on Temporal schedule, decoupled from provider activity |

Where the heartbeat does apply (GKE resources), useful signals it enables:
- Per-resource "last observed at" surfaced from informer event timestamps
- Health check: if `now - lastObservedAt > 2 × pollInterval`, the provider may be stuck

**Implication for drift re-detection:** Once drift is detected and `Synced=False` stabilizes:

- **Approach A:** Re-detects on every poll regardless. At `DRIFT_POLL_INTERVAL=30s`, it logs
  `DRIFT DETECTED` repeatedly. Temporal workflow ID dedup prevents duplicate workflows.
- **Approach C:** Re-fires only when `resourceVersion` changes. For GKE (unconditional writes),
  re-detection is continuous. For CloudSQL/Compute/Storage, drift may be detected once and go
  silent until conditions actually change again — meaning a missed Temporal start would not
  self-recover from the watcher side alone.

**Implication for K8s API load (corrects Metric 9 for Approach C):** C's load is not uniformly
"bounded by the provider poll rate". For `provider-upbound-gcp-container`, it fires on every
poll (same rate as provider). For all other providers, C is completely idle in steady state —
zero calls beyond the persistent watch stream. Approach A generates LIST calls on its own
schedule regardless of provider behavior.

---

### F-03: Not all `managementPolicies` combinations are valid — only specific sets are supported

**Observed during:** Drift recovery testing — attempting to flip a `DatabaseInstance` from
`["Observe"]` back to an active management mode.

Setting `spec.managementPolicies: ["Observe", "Update"]` was rejected immediately:

```
`spec.managementPolicies` is set to a value([Observe Update]) which is not supported.
Check docs for supported policies  reason=ReconcileError  status=False  type=Synced
```

The error surfaces at reconcile time, not at admission — the policy is written to etcd
successfully but Crossplane rejects it when it tries to act on it.

**Known valid combinations (empirically confirmed):**

| Value | Meaning | Use case |
|---|---|---|
| `["*"]` | Full management — Create, Observe, Update, Delete | Default full management |
| `["Observe"]` | Read-only — no create, update, or delete | Drift protection mode |
| `["Create", "Observe"]` | Create if missing, then observe — no update or delete | Recover deleted resources without touching field drift |
| `["Create", "Observe", "Update"]` | Create and update but no delete | Full drift recovery without deletion risk |

**Impact on `FlipManagementPolicyActivity`:** The choice of target policy determines what
Crossplane will reconcile during the recovery window:

- Use `["Create", "Observe"]` when the drift signal is deletion only (`Synced=False/ReconcileError`)
  and field changes should not be overwritten.
- Use `["Create", "Observe", "Update"]` for full drift recovery (both deletion and field drift).
- Use `["*"]` only if deletion of the resource by Crossplane is also acceptable.
- Always flip back to `["Observe"]` after recovery completes.

---

### F-04: Approach C fires events during recovery — `DRIFT_CLEAN` confirms reconciliation progress

**Observed during:** Drift recovery test on approach C with `DRIFT_VERBOSE=true`.

**Concern:** Since approach C is event-driven, there was a question of whether it would go
silent after drift was first detected — providing no visibility into whether recovery actually
succeeded, unlike approach A which re-detects on every poll.

**What was actually observed:** Recovery produces a clear sequence of events:

1. **Management policy flip** (`spec.managementPolicies` changes from `["Observe"]` to
   `["Create", "Observe", "Update"]`) — this is a spec write, which fires a `MODIFIED` event
   for all provider types → `DRIFT_CLEAN` logged (drift condition is now gone or changing)
2. **Crossplane reconciling** — as Crossplane creates or reverts the GCP resource, it updates
   `status.conditions` repeatedly (Synced=False/Creating → Synced=True, Ready=False → Ready=True)
   — each condition write fires a `MODIFIED` event → `DRIFT_CLEAN` logged on each
3. **Reconcile complete** — final condition update to Synced=True/Ready=True fires a last
   `MODIFIED` → `DRIFT_CLEAN` logged
4. **Stable** — silence (for non-container providers in steady state)

**Key insight:** Condition changes fire `MODIFIED` events for all provider types — not just
`provider-upbound-gcp-container`. The F-02 finding (unconditional writes are container-provider-
specific) applies to steady-state heartbeat only. During active reconciliation, every provider
writes conditions, so approach C receives events throughout the full recovery window regardless
of provider type.

**Recovery visibility comparison:**

| Signal | A | C |
|--------|---|---|
| Drift present | `DRIFT DETECTED` every poll | `DRIFT DETECTED` on first event |
| Recovery in progress | `DRIFT DETECTED` continues while `Synced=False/ReconcileError` | `DRIFT_CLEAN` on each condition change (verbose) |
| Recovery complete | Silence — no log on clean poll (by design, to avoid noise) | Silence — no more events with drift conditions |
| Stable (no drift) | Silence | Silence (non-container) or `DRIFT_CLEAN` every poll (container) |

Both approaches use silence to signal recovery completion — neither logs an explicit "drift
resolved" entry. The difference is in the recovery window: A keeps re-detecting as long as
`Synced=False/ReconcileError` is present (every poll); C emits `DRIFT_CLEAN` events as
conditions change during reconciliation (visible only with `DRIFT_VERBOSE=true`), giving a
live trace of the recovery progression that A does not provide.

---

### F-05: Approach D scan cost is negligible — Temporal workflow history is a free audit log

**Observed during:** Approach D testing — running `DriftScanWorkflow` manually and via schedule.

**Scan cost breakdown:**

| Factor | Cost | Notes |
|--------|------|-------|
| K8s API | Negligible | One LIST per GVR per scan, served from the API server's watch cache — not a direct etcd read |
| Memory | Negligible | Activity receives the LIST response, iterates, discards — periodic spike then GC'd, no persistent cache |
| CPU | Negligible | Simple map diff per MR |
| Temporal DB | Grows at `scan_rate × retention_days` | The only cost that accumulates — managed with Temporal's retention policy |

With 5 GVRs at 1-min interval: 5 LIST calls/min, ~1,440 workflow executions/day. Each run is a
single small activity returning a count struct. At POC scale this is negligible. In production,
configure a Temporal retention policy (e.g. 30 days) to bound DB growth.

**Workflow history as audit log:**

Every `DriftScanWorkflow` execution records in Temporal's event history:

- **What was scanned** — `ScanDriftInput` with GVR list
- **What drifted** — `ScanDriftOutput` with `scannedResources` and `driftedResources` counts, plus
  `DRIFT DETECTED` log entries in the activity event history
- **When** — workflow start timestamp
- **Outcome** — activity result, any errors

This is true for **all drift action modes** — even `notify` mode where no `DriftApprovalWorkflow`
is started, the scan workflow itself records that drift was found. The Temporal UI shows the full
history queryable by time, workflow ID, and status.

**Contrast with Approaches A and C:** Detection happens outside Temporal in both. For `notify` mode
(no approval workflow started), there is no Temporal record of the drift event — only pod logs.
Any audit trail requires a separate log aggregator or DB write. Approach D gets a structured,
queryable audit log for free by virtue of the scan being a workflow.

---

### F-06: Approach D POC worker deployment conflicts with Temporal's internal worker service

**Observed during:** Approach D testing — Temporal schedules were created but never fired.

---

#### Issue 1: Deployment name collision kills Temporal's internal worker

The Temporal Helm chart deploys four server-side services: `temporal-frontend`, `temporal-history`,
`temporal-matching`, and `temporal-worker`. The last one — `temporal-worker` — is Temporal's own
internal worker service. It runs Temporal's system workflows, including the **schedule management
workflow** that fires `DriftScanWorkflow` on the configured interval.

`k8s/temporal-worker/deploy.sh` runs:

```bash
kubectl delete deployment temporal-worker -n temporal-system --ignore-not-found=true
kubectl apply -k .
```

`k8s/temporal-worker/deployment.yaml` uses `name: temporal-worker` — the same name as Temporal's
internal service. The deploy script explicitly deletes the existing deployment before applying,
which means every `make deploy-worker` or `make dev` run silently kills Temporal's internal worker
and replaces it with the app binary.

**Consequence:** Schedules are registered in Temporal's database but never executed. The matching
service logs show `temporal-sys-per-ns-tq` (the queue the schedule system workflow runs on) going
idle immediately — no consumer is running. `temporal schedule trigger` acknowledges the request but
no workflow execution appears.

**Fix:** Rename the app worker deployment to avoid collision:

- `k8s/temporal-worker/deployment.yaml` — `name: temporal-worker` → `name: ucp-worker`
- `k8s/temporal-worker/serviceaccount.yaml` — all references to `temporal-worker` → `ucp-worker`
- `k8s/temporal-worker/configmap.yaml` — `name: temporal-worker-config` → `ucp-worker-config`
- `k8s/temporal-worker/deploy.sh` — update `kubectl delete` and label selectors to `ucp-worker`

After the rename, Temporal's internal `temporal-worker` deployment is left intact and schedules
fire correctly.

---

#### Issue 2: Drift detection merged into the provisioning worker binary

`cmd/worker/main.go` registers `DriftScanWorkflow` and `ScanDriftActivity` alongside all
provisioning workflows (`RequestDatabaseWorkflow`, `RequestComputeWorkflow`, etc.) in a single
binary, polling a single task queue (`db-provisioning`). This was a POC shortcut.

**Problems with this in production:**

- Drift detection and provisioning have different resource profiles, scaling requirements, and
  deployment cadences — they should be independently deployable
- A crash or redeploy of the provisioning worker kills drift scanning and vice versa
- Drift scanning for a schedule-based approach should poll its own task queue (`drift-detection`),
  not share one with provisioning workflows

**Correct production structure:**

```
backend/temporal-worker/
  cmd/
    provisioning-worker/main.go   ← registers provisioning workflows, polls db-provisioning
    drift-worker/main.go          ← registers DriftScanWorkflow + DriftApprovalWorkflow, polls drift-detection
  internal/
    workflows/                    ← shared — imported by both
    activities/                   ← shared — imported by both

k8s/
  provisioning-worker/
    deployment.yaml               ← image: provisioning-worker, TEMPORAL_TASK_QUEUE: db-provisioning
    configmap.yaml
    deploy.sh
  drift-worker/
    deployment.yaml               ← image: drift-worker, TEMPORAL_TASK_QUEUE: drift-detection
    configmap.yaml
    deploy.sh
```

Both binaries import shared types and activities from `internal/`. The Temporal schedule's
`--task-queue` is set to `drift-detection` to match the drift worker exclusively.

---

### F-07: Some drifts are non-recoverable — Crossplane cannot reverse all GCP changes

**Observed during:** Approach D drift approval workflow testing (MCUCP-158 implementation).

Certain GCP-side changes cannot be automatically reversed by Crossplane even after management
policies are restored to `["Create", "Observe", "Update"]`. The recovery approval workflow
completes, Crossplane attempts reconciliation, and GCP refuses the update. The workflow
correctly times out and flips back to `["Observe"]`, but the drift remains.

**Confirmed non-recoverable cases:**

**Case 1 — Immutable field modification (CloudSQL disk size increase)**

GCP Cloud SQL does not allow decreasing `settings.disk_size`. When a disk is manually increased
in the GCP Console (e.g. 50 GB → 150 GB), approving drift recovery causes Crossplane to attempt
setting `disk_size` back to 50 GB. GCP rejects this:

```
update failed: async update failed: refuse to update the external resource because the
following update requires replacing it: cannot change the value of the argument
"settings.0.disk_size" from "150" to "50"
```

The MR condition: `Synced=False, reason=ReconcileError, type=LastAsyncOperation: AsyncUpdateFailure`.

This persists indefinitely — `isDrifted()` keeps returning true, `WaitMRReadyActivity` times out
after 30 min (or now after the grace period with `RECONCILE_FAILED`), R3 flips back to Observe.

**Case 2 — Resource deletion with stale late-initialized fields (GKE Cluster / NodePool)**

See F-01 for the full failure sequence. Summary: when a GKE Cluster or NodePool is deleted from
GCP and Crossplane attempts recreation, late-initialized fields in `forProvider` reference
state that belonged to the deleted resource (pod secondary range names, deprecated
`loggingService` + new `loggingConfig` conflict, etc.). Crossplane's recreate fails
permanently; the drift cannot be auto-resolved.

**General pattern — when auto-recovery fails:**

| Root cause | Examples | Recoverable? |
|---|---|---|
| GCP enforces immutable fields | `disk_size` reduction, zone change, database version downgrade | ❌ Requires manual change or resource replacement |
| Late-initialized `forProvider` references deleted state | GKE pod secondary ranges, deprecated field conflicts | ❌ Requires MR deletion and XR-level recreation |
| Field requires resource replacement | Changing CloudSQL `databaseVersion` to a lower version | ❌ GCP only supports version upgrades in-place |
| Transient GCP error | API quota, temporary outage | ✅ Retries on next scan cycle after recovery |

**Impact on `WaitMRReadyActivity`:**

When non-recoverable drift is detected, the activity now:
1. Waits a 2-minute grace period (allows Crossplane to start reconciling)
2. If `Synced=False/ReconcileError` persists past the grace period → returns early with the
   GCP error message (type: `RECONCILE_FAILED`, non-retryable)
3. R3 (`FlipManagementPolicyActivity` back to `["Observe"]`) always runs regardless

The operator receives the exact GCP error message in the Temporal UI and can decide
whether to accept the drift, manually fix the GCP resource, or delete and recreate.

**Not handled in this POC — future considerations:**

- Detecting specific non-recoverable conditions early and surfacing them before approval
  (so the operator knows recovery is impossible before approving)
- Automated deletion + recreation path for deletion drift with stale `forProvider`
  (see F-01 recommendation)
- Distinguishing "immutable field change" from "transient failure" in the notification
  to give operators clearer recovery guidance

---

## Post-POC Design Considerations

> Out of scope for this POC. Captured here for the production design discussion.

---

### Drift action modes

Two modes should be supported per resource, controlled via XRD parameter:

| Mode | Behaviour |
|---|---|
| `notify` | Detect drift, send notification, no approval workflow, no reconciliation |
| `approve` | Detect drift, send notification, wait for human approval, reconcile if approved |

Propagated as a label on the composed MR by the composition:

```
platform.io/drift-protection: "true"
platform.io/drift-action: "notify" | "approve"
```

---

### The chatty notification problem

Drift is **continuously re-detected** on every poll cycle (A, D) or on every provider write (B, C). Without a debounce mechanism, the same drift event generates a notification every 30s–1min indefinitely.

**For `approve` mode** — already solved. Temporal's workflow ID dedup (`AlreadyStartedError`) suppresses re-fires as long as the approval workflow is running. After rejection or timeout, a cooldown is needed before the workflow terminates so re-detection is suppressed during that window.

**For `notify` mode** — Temporal is not involved, so a separate snooze mechanism is required.

---

### Snooze mechanism

Store the snooze state as an **annotation on the MR itself**, not in a separate database:

```
platform.io/drift-snoozed-until: "2026-04-20T11:30:00Z"
```

**Why annotation and not a DB column:**
The MR object is already fetched in the watcher's `List()` call. Checking the annotation costs nothing — it is already in memory. A DB read per MR per poll cycle would be N × poll_rate queries, which is expensive at scale.

**Why annotation and not a label:**
Labels are indexed in etcd and intended for selection. A high-cardinality timestamp value is a bad fit. Annotations carry metadata and have no indexing concern.

**Default snooze: 30 minutes.** Configurable per tenant in the tenant config table.

---

### Who writes the snooze annotation

| Mode | Written by | When |
|---|---|---|
| `notify` | Watcher (directly after notification fires) | Immediately after sending notification |
| `approve` | Temporal `SetSnoozeActivity` | On workflow rejection or timeout |

For `approve` mode, clearing the snooze annotation after a successful reconciliation is handled by the same activity (`SetSnoozeActivity(duration=0)`).

The watcher only needs PATCH permission on MR annotations — it remains read-only for everything else.

---

### User controls (UI / CLI)

Snooze state is just a K8s annotation — the UI patches it directly via the K8s API:

```bash
# Extend snooze
kubectl annotate mr <name> platform.io/drift-snoozed-until="<timestamp>" --overwrite

# Cancel snooze immediately (re-arms on next poll)
kubectl annotate mr <name> platform.io/drift-snoozed-until-

# Disable drift protection entirely
kubectl label mr <name> platform.io/drift-protection=false --overwrite
```

No custom backend endpoint needed. The web UI wraps the same K8s API calls.

---

### State storage summary

| Store | Owns | Notes |
|---|---|---|
| **etcd** (via K8s API) | Drift signal (`forProvider`/`atProvider`) + snooze annotation | Read for free in existing `List()` call |
| **DB** | Audit log | Written async (fire-and-forget) from both watcher and Temporal activities |
| **Temporal** | Approval workflow state | `approve` mode only |

---

## Summary

### Winner: Approach D

Approach D (Temporal Schedule) is the clear winner with a total score of **37/44** — ahead
of A and C (both 32) and B (27).

Key reasons D wins:
- **No new infrastructure.** The Temporal worker already exists. Drift scanning is just a
  new workflow + activity registered in the same binary.
- **Richest observability out of the box.** Every scan run is a Temporal workflow with
  structured output (scanned/drifted counts) visible in the UI, queryable by time and status.
  A free audit log with zero extra tooling.
- **Best natural scale-out.** Temporal distributes activity execution across the worker fleet
  automatically. No manual pod-to-GVR mapping, no sharding strategy to design.
- **Production-ready today.** No alpha dependencies, no new languages, no new K8s objects.

### Future: Approach B

Once WatchOperations graduates from alpha, **Approach B is technically superior** on the
metrics that matter for a platform at scale:

- **Event-driven with sub-second reaction time** after Crossplane updates `atProvider` — no
  poll delay.
- **No persistent process.** Operation pods are ephemeral — no heap accumulation, no crash
  recovery strategy needed.
- **Lowest K8s API load.** Idle watch streams instead of periodic LIST calls.

The only blocker is the alpha flag. When the feature graduates, B becomes the natural
replacement for D with minimal code changes.

### Interchangeability

All four approaches share the same detection logic at the core — two complementary signals:

1. `forProvider vs atProvider` field diff
2. `.status.conditions[Synced]=False/ReconcileError`

The detection layer is a thin wrapper around these two signals. Swapping from one approach to
another means changing **how** the wrapper is triggered (poll timer, Kubernetes watch stream,
Temporal schedule, WatchOperation), not **what** it checks.