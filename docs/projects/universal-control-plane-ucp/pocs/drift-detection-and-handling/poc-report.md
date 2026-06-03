---
title: "POC Report — Drift Detection and Handling"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Drift Detection and Handling

| | |
|---|---|
| **Jira** | [MCUCP-158](https://jira.rakuten-it.com/jira/browse/MCUCP-158) |
| **Author** | aripermana.putra |
| **Date** | 2026-06-02 |
| **Status** | COMPLETED |

---

## 1. Summary

MCUCP-158 proves that UCP can detect configuration drift on GCP-managed resources and
orchestrate a human approval workflow before reconciliation. Four trigger approaches were
evaluated and benchmarked. Approach D (Temporal Schedule) was selected. A shared
detection and recovery design underpins all four approaches — switching trigger mechanism
requires no changes to detection logic or the approval workflow.

**Verdict: Go.** Drift detection and the human approval workflow are functionally
complete. Approach D has no alpha dependencies and requires no new infrastructure —
the clearest path to production among the four. Several design decisions remain open
before production hardening.

---

## 2. Objectives & Success Criteria

**Hypothesis:**
UCP can detect when a cloud resource's actual state diverges from the desired state
declared in Crossplane, and orchestrate a human approval flow before allowing
Crossplane to reconcile the drift.

**Success criteria:**

| # | Criterion | Result |
|---|---|---|
| SC-1 | Field-level drift detected via `forProvider` vs `atProvider` diff | Pass |
| SC-2 | Resource deletion drift detected via `Synced=False/ReconcileError` | Pass |
| SC-3 | `DriftApprovalWorkflow` fires on drift detection | Pass |
| SC-4 | Temporal workflow ID dedup prevents duplicate workflows for the same drifted resource | Pass |
| SC-5 | Management policy flip unlocks Crossplane reconciliation on approval | Pass |
| SC-6 | Management policy is always restored to `["Observe"]` after recovery, regardless of outcome | Pass |
| SC-7 | Non-recoverable drift correctly times out and surfaces the GCP error to the operator | Pass |
| SC-8 | Approach D (Temporal Schedule) achieves the highest benchmark score across all evaluated metrics | Pass |

**Scope boundaries (out of scope):**
- Real notification delivery (Slack, PagerDuty) — notification stub implemented; swap in production
- Omnia (`provider-roc`) drift support — requires `atProvider` changes in provider-roc
- Drift detection for Terraform-managed resources

---

## 3. Findings

### Two complementary detection signals are required

Drift manifests in two distinct ways, each requiring a different signal:

| Drift type | Detection signal | Why the other signal fails |
|---|---|---|
| Field change | `forProvider` vs `atProvider` diff | `Synced` may remain `True` — diff is the only signal |
| Resource deletion | `Synced=False, reason=ReconcileError` | `atProvider` retains stale values after deletion — diff alone misses this |

Both signals must be checked. Neither alone is sufficient.

### Approach D selected — Temporal Schedule

All four approaches were benchmarked across 11 metrics (detection latency, setup
complexity, code volume, extensibility, debuggability, observability, failure resilience,
deduplication, K8s API load, production readiness, scalability):

| Approach | Score | Key characteristic |
|---|---|---|
| D — Temporal Schedule | **37/44** | Best scalability, richest observability, no new infrastructure |
| A — Polling Watcher | 32/44 | Simple but requires a separate deployment |
| C — Informer Watcher | 32/44 | Event-driven but high memory at scale |
| B — WatchOperations | 27/44 | Superior when alpha graduates; blocked on alpha flag today |

All approaches share the same ~1 min detection floor — the Crossplane provider poll
interval is the bottleneck, not the trigger mechanism.

### All approaches are interchangeable

All four share the same detection logic (two complementary signals) and the same
`DriftApprovalWorkflow`. Switching trigger mechanism means changing only how the wrapper
is triggered — not what it checks. Approach D can be replaced by Approach B with minimal
code changes once WatchOperations graduates.

### Informer heartbeat is specific to `provider-upbound-gcp-container`

Approach C's heartbeat benefit — detecting whether the provider is actively observing
a resource — only applies to GKE resources managed by `provider-upbound-gcp-container`.
That provider calls `Status().Update()` unconditionally on every reconcile cycle,
causing unconditional `resourceVersion` increments. The other providers (`sql`,
`compute`, `storage`) do not write unless something changed, so they are completely
silent in steady state. The heartbeat does not generalize.

### Some drifts are non-recoverable

Certain GCP-side changes cannot be automatically reversed by Crossplane even after
management policies are restored:

| Root cause | Examples | Recoverable? |
|---|---|---|
| GCP enforces immutable fields | `disk_size` reduction, zone change, database version downgrade | ❌ Requires manual intervention or resource replacement |
| Late-initialized `forProvider` references deleted state | GKE pod secondary ranges, deprecated + new logging field conflict | ❌ Requires MR deletion and XR-level recreation |
| Transient GCP error | API quota, temporary outage | ✅ Retries on next scan cycle |

When non-recoverable drift is detected, the approval workflow surfaces the exact GCP
error message, times out `WaitMRReadyActivity`, and always restores `["Observe"]` via R3
— the MR is never left in full management mode.

### Only specific `managementPolicies` combinations are valid

`["Observe", "Update"]` is rejected by Crossplane at reconcile time (not at admission).
Valid combinations confirmed empirically:

| Value | Use case |
|---|---|
| `["Observe"]` | Drift protection mode — read-only |
| `["Create", "Observe"]` | Recover deletion without touching field drift |
| `["Create", "Observe", "Update"]` | Full drift recovery |
| `["*"]` | Full management including deletion |

### Approach D observability is a free audit log

Every `DriftScanWorkflow` execution records what was scanned, what drifted, and when —
visible in Temporal UI, queryable by time and status. For `notify` mode (no approval
workflow started), Approaches A and C produce no Temporal record — only pod logs.
Approach D gets a structured, queryable audit log for free.

---

## 4. Open Questions

1. **Chatty notification problem** — drift is continuously re-detected on every scan
   cycle. Without a debounce mechanism, the same drift event generates repeated
   notifications. For `approve` mode, Temporal dedup already solves this. For `notify`
   mode, a snooze mechanism is needed. The design (annotation on the MR:
   `platform.io/drift-snoozed-until`) is in the shared design doc — the question is
   how snooze duration should be configured and who controls it.

2. **How to handle non-recoverable drift** — certain drifts are confirmed non-recoverable:
   immutable field changes (e.g. `disk_size` reduction) and deletion with stale
   late-initialized `forProvider` fields (e.g. GKE pod secondary ranges). Detection and
   classification of non-recoverable cases is testable and automated. The open question
   is what UCP should do when it detects one: surface early before approval so the
   operator knows reconciliation will fail, or accept the current behaviour (approve →
   timeout → error surfaces) and invest in better error messaging instead.

---

## 5. Recommendations

**Decision: Go**

Approach D has no alpha dependencies and requires no new infrastructure — the clearest
path to production. The detection logic and approval workflow are solid and shared across
all approaches, making a future migration to Approach B straightforward when WatchOperations
graduates.

**Risks:**

| Risk | Severity | Mitigation |
|---|---|---|
| Scan cost scales with resource count — each scan cycle iterates all drift-protected MRs | Medium | Two-phase workflow: `DiscoverMRsActivity` per GVR + `ScanTenantActivity` per `(GVR, tenant)`. Load distributed across worker pods automatically via Temporal task queue. Autoscale workers with KEDA. |
| Scan overlap / race condition — if a scan takes longer than the interval, the next scan starts before the previous completes | Medium | Temporal schedule overlap policy `SKIP` — next scan only starts after `DriftScanWorkflow` fully completes (all activities done). Workflow is the natural fence. |
| False positives from late-initialized `forProvider` fields — GCP writes observed values back into `forProvider`, causing persistent diffs that are not real drift | Medium | Skip list for known late-initialized fields (e.g. `bootDisk`, `clusterRef`). Skip list grows as more resource types are added — needs maintenance process as new providers are onboarded. |
| Provider poll interval is the actual detection floor — drift is only detectable after Crossplane's `Observe()` updates `atProvider` | Low | Separate concern from scan frequency. Monitor provider pod health and poll lag independently. Configured at `--poll=1m` in PoC. |

**Next steps:**

1. Decide on snooze duration configuration and ownership (Open Question 1)
2. Identify most if not all non-recoverable drift scenarios across all resource types
   per cloud provider, then decide on handling strategy — early detection vs better
   error messaging (Open Question 2)

---

## 6. References

- Design docs: [Shared Design](./shared-design.md) · [POC Results](./poc-results.md)
- Reconciliation limitations: [reconciliation-limitations.md](./reconciliation-limitations.md)
- PRs: [ucp-platform #55](https://ghe.rakuten-it.com/clsd-ucp/ucp-platform/pull/55) · [ucp-api-gateway #18](https://ghe.rakuten-it.com/clsd-ucp/ucp-api-gateway/pull/18)
- Jira: [MCUCP-158](https://jira.rakuten-it.com/jira/browse/MCUCP-158)
