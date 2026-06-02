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
complete. Approach D is production-ready today. Several design decisions remain open
for the production hardening phase.

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
- Periodic background sync (login-triggered only in PoC)
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

1. **Notify vs approve mode per resource** — two modes are needed: `notify` (detect,
   alert, no reconciliation) and `approve` (detect, alert, wait for human approval,
   reconcile). The propagation mechanism (XRD parameter + MR label) is designed but
   not fully implemented.

2. **Chatty notification problem** — drift is continuously re-detected on every scan
   cycle. Without a debounce mechanism, the same drift event generates repeated
   notifications. For `approve` mode, Temporal dedup already solves this. For `notify`
   mode, a snooze mechanism is needed (designed: annotation on the MR,
   `platform.io/drift-snoozed-until`; not implemented).

3. **Non-recoverable drift — early detection** — currently the operator only learns
   a drift is non-recoverable after approving and waiting for the timeout. Whether
   to detect specific non-recoverable conditions (immutable fields, stale late-init)
   before approval and surface them proactively needs a decision.

4. **Late-initialized `forProvider` for deletion drift** — when a resource is deleted
   and Crossplane recreates it, stale late-initialized `forProvider` fields cause
   recreation to fail. The correct approach (delete composed MRs before switching
   management policy) needs to be implemented for the deletion recovery path.

5. **Omnia drift support** — `provider-roc`'s `OmniaDatabaseObservation` struct
   only contains connection info. It must be extended to mirror driftable `forProvider`
   fields so the `forProvider` vs `atProvider` diff can work for Omnia resources.

6. **Approach B (WatchOperations) migration path** — Approach B is technically superior
   on detection latency, memory footprint, and K8s API load. The migration plan from
   Approach D to B when WatchOperations graduates from alpha should be defined.

---

## 5. Recommendations

**Decision: Go**

Approach D is production-ready with no alpha dependencies, no new infrastructure, and
the richest built-in observability. The detection logic and approval workflow are solid
and shared across all approaches, making a future migration to Approach B straightforward.

**Next steps:**

1. Implement notify vs approve mode and the snooze mechanism (Open Questions 1, 2)
2. Implement the deletion recovery path — delete composed MRs before policy flip for
   deletion drift (Open Question 4)
3. Extend `provider-roc` `atProvider` to support drift detection for Omnia resources
   (Open Question 5)
4. Define the Approach B migration plan for when WatchOperations graduates (Open Question 6)

---

## 6. References

- Design docs: [Shared Design](./shared-design.md) · [POC Results](./poc-results.md)
- Reconciliation limitations: [reconciliation-limitations.md](./reconciliation-limitations.md)
- PRs: [ucp-platform #55](https://ghe.rakuten-it.com/clsd-ucp/ucp-platform/pull/55) · [ucp-api-gateway #18](https://ghe.rakuten-it.com/clsd-ucp/ucp-api-gateway/pull/18)
- Jira: [MCUCP-158](https://jira.rakuten-it.com/jira/browse/MCUCP-158)
