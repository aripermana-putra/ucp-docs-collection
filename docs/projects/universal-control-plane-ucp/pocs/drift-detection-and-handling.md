---
title: "Drift Detection and Handling"
space: UCP
parent_page_id: "../pocs.md"
---

# Drift Detection + Human Approval — POC Overview

## Goal

Prevent Crossplane from silently auto-healing infrastructure drift. When external state
diverges from desired state, block reconciliation, notify operators, and wait for human
approval before proceeding.

**Scope:** All Crossplane-managed resource types (CloudSQL, GCE Compute, GKE, GCS, future
load balancers, VPCs, etc.) — not just databases. The POC is validated on CloudSQL first,
but the design is generic from day one.

---

## End-to-End Flow

This is the target behaviour for all four approaches. The detection mechanism differs; everything from the trigger condition onward is identical.

### Trigger Condition

A resource is considered drifted when its XR has **all** of:

```
XR.status.conditions[Synced].status == "False"
AND
XR.status.conditions[Synced].reason == "ReconcileError"
```

`ReconcilePaused` is set by the **pause annotation** (`crossplane.io/paused: "true"`), not by `managementPolicies: ["Observe"]` — the watcher ignores it. With `managementPolicies: ["Observe"]` and a healthy resource, Crossplane sets `Synced=True/ReconcileSuccess`.

### Full Flow

```
GCP resource drifted (deleted, modified outside Crossplane)
        │
        │  ~1 min  — provider poll interval (--poll=1m, Crossplane default)
        ▼
[Provider reconciler]
  Calls GCP API → 404 / mismatch
  Sets MR (e.g. DatabaseInstance): Synced=False, reason=ReconcileError
        │
        │  seconds — composite reconciler watches MR via K8s watch stream
        ▼
[Composite reconciler]
  Aggregates MR conditions → XR: Synced=False, reason=ReconcileError
        │
        │  seconds (B/C) / up to 30s (A) / up to 1 min (D)
        │  — depends on approach detection mechanism
        ▼
[Drift watcher / WatchOperation / Temporal schedule]
  Detects XR: Synced=False, reason=ReconcileError
  Starts DriftApprovalWorkflow (deduped by workflow ID)
        │
        ▼
NotifyDriftActivity
  → structured log entry (POC stub)
  → swap point for Slack + email + PagerDuty in production
        │
        ▼
Wait for "approval-signal" — 24h timeout
        │
        ├─ APPROVED ──────────────────────────────────────────┐
        │                                                      ▼
        │                              FlipManagementPolicyActivity → policy="*"
        │                              WaitXRReadyActivity (30 min timeout)
        │                              FlipManagementPolicyActivity → policy="Observe"
        │                              (R3 always runs — safety: never leave resource unmanaged)
        │
        └─ REJECTED or TIMEOUT
                              do nothing — workflow terminates
                              resource stays in Observe mode (no auto-heal)
```

### Why `managementPolicies: ["Observe"]` on the MR

When `driftProtection: true` is set on the XR, the composition renders `managementPolicies: ["Observe"]` on each composed managed resource (e.g. `DatabaseInstance`). This tells Crossplane: observe the resource, detect drift, report it — but **do not reconcile it**. The resource stays drifted until a human approves.

`FlipManagementPolicyActivity` patches the MR's `managementPolicies` field directly. Setting it to `["*"]` re-enables full management so Crossplane can reconcile. After `WaitXRReady`, it flips back to `["Observe"]`.

---

## Four Approaches Under Evaluation

| | Approach | Branch | Trigger mechanism |
|-|----------|--------|-------------------|
| A | Go Polling Watcher | `feature/drift-poc-approach-a-watcher` | Periodic poll of XR conditions |
| B | WatchOperations + function-python | `feature/drift-poc-approach-b-watchoperations` | Crossplane-native event trigger |
| C | Informer-based Watcher | `feature/drift-poc-approach-c-informer` | K8s watch stream (event-driven) |
| D | Temporal Schedule | `feature/drift-poc-approach-d-temporal-schedule` | Temporal cron scan |

All branches base off: `feature/crossplane-v2-temporal-worker-migration`

---

## Shared Design Decisions (apply to all approaches)

See [Shared Design](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6591418173) — all four approaches build on it.

Key decisions:

1. **Watch at XR level**, not MR level — works for all resource types generically
2. **Label-based opt-in** — `platform.io/drift-protection: "true"` on any XR, no XRD
   changes required for targeting
3. **XRD `driftProtection` param** — controls whether the composition renders
   `managementPolicies: ["Observe"]` on composed managed resources
4. **Generic `DriftApprovalWorkflow`** — carries `xrGroup/Version/Resource/Kind` so it
   works for any XR type
5. **Generic `WaitXRReadyActivity`** — replaces per-type wait activities
6. **Notification stub** — `NotifyDriftActivity` logs a structured entry; swap point for
   Slack + email + PagerDuty in production

---

## Child Pages

| Page | Contents |
|------|----------|
| [Shared Design](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6591418173) | Common types, workflow, activities, Crossplane changes |
| [Approach A — Polling Watcher](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6587351085) | Go binary polling XR conditions |
| [Approach B — WatchOperations](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6591353112) | Crossplane WatchOperations + function-python |
| [Approach C — Informer Watcher](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6587351087) | K8s informer-based event-driven watcher |
| [Approach D — Temporal Schedule](https://confluence.rakuten-it.com/confluence/pages/viewpage.action?pageId=6591418235) | Temporal cron schedule scan |
