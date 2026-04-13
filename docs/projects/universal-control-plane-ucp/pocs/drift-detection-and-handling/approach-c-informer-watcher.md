---
title: "Approach C — Informer Watcher"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Approach C — Informer-based Watcher

**Branch:** `feature/drift-poc-approach-c-informer`
**Trigger mechanism:** K8s shared informer watch stream (event-driven, not polling)

---

## How It Works

A Go binary (`drift-watcher`) — same concept as Approach A — but instead of periodic
list-polls, it uses `client-go` **shared informers** that maintain a persistent K8s watch
stream per XR type. When any watched XR's conditions change, the informer pushes the event
immediately to an event handler. No polling delay.

```
K8s API server (persistent watch stream, one per GVR)
    └── informer pushes event when XR object changes
            └── event handler runs:
                  if isDrifted(xr): fireWorkflow(xr)
                  else: no-op
```

This is the production-grade evolution of Approach A. Same binary structure, same Temporal
integration — event-driven instead of poll-driven.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Kubernetes                                              │
│                                                         │
│  XR objects with label platform.io/drift-protection    │
│    │                                                    │
│    │  K8s watch stream (persistent HTTP/2 connection)  │
│    ▼                                                    │
│  SharedInformerFactory                                  │
│  ├── informer for GVR: xdatabases                      │
│  ├── informer for GVR: xcomputeinstances               │
│  ├── informer for GVR: xkubernetesclusters             │
│  └── informer for GVR: xobjectstorages                 │
│       └── OnUpdate / OnAdd handler (same for all GVRs) │
│             if isDrifted → fireWorkflow(xr)             │
└──────────────────────────┬──────────────────────────────┘
                           │  Temporal Go SDK
                           ▼
              DriftApprovalWorkflow
              (see Shared Design)
```

---

## Multi-Resource Support

Informers are set up per GVR, all sharing the same event handler. Same ConfigMap-driven
GVR list as Approach A:

```go
for _, gvr := range configuredGVRs {
    inf := factory.ForResource(gvr).Informer()
    inf.AddEventHandler(cache.ResourceEventHandlerFuncs{
        UpdateFunc: func(_, new interface{}) { onXREvent(new, gvr, tc) },
        AddFunc:    func(obj interface{})    { onXREvent(obj, gvr, tc) },
    })
}
```

Adding a new resource type = add one line to the ConfigMap. No code change, no redeployment.

---

## Key Code: drift-watcher/main.go

```go
func main() {
    tc, _ := client.Dial(client.Options{HostPort: getenv("TEMPORAL_ADDRESS", "localhost:7233")})
    dc, _ := k8s.NewDynamicClient()
    gvrs  := parseGVRs(getenv("XR_GVRS", ""))
    stopCh := make(chan struct{})

    factory := dynamicinformer.NewFilteredDynamicSharedInformerFactory(
        dc,
        10*time.Minute,   // resync period — safety net for missed events
        metav1.NamespaceAll,
        func(opts *metav1.ListOptions) {
            opts.LabelSelector = "platform.io/drift-protection=true"
        },
    )

    for _, gvr := range gvrs {
        gvr := gvr
        inf := factory.ForResource(gvr).Informer()
        inf.AddEventHandler(cache.ResourceEventHandlerFuncs{
            UpdateFunc: func(_, newObj interface{}) {
                xr := newObj.(*unstructured.Unstructured)
                if isDrifted(xr) {
                    fireWorkflow(context.Background(), tc, buildDriftInput(xr, gvr))
                }
            },
            // AddFunc: catches resources already drifted at watcher startup
            AddFunc: func(obj interface{}) {
                xr := obj.(*unstructured.Unstructured)
                if isDrifted(xr) {
                    fireWorkflow(context.Background(), tc, buildDriftInput(xr, gvr))
                }
            },
        })
    }

    factory.Start(stopCh)
    factory.WaitForCacheSync(stopCh)
    log.Printf("informers synced, watching %d GVRs", len(gvrs))
    <-stopCh
}
```

---

## Resync Period

The informer factory uses a **10-minute resync period**:
- Every 10 minutes, all watched resources are re-listed and `UpdateFunc` fires for each.
- Serves as a safety net: any event missed during a transient network blip is picked up.
- `isDrifted()` is idempotent; `fireWorkflow()` is deduplicated via Temporal workflow ID.

This gives Approach C both event-driven speed and polling safety.

---

## Comparison with Approach A

| Dimension | Approach A (polling) | Approach C (informer) |
|-----------|---------------------|----------------------|
| Trigger latency after Synced=False | 0–30s | <1s |
| K8s API calls (idle, 50 resources) | ~8 list calls/min | 4 persistent watch streams |
| Safety on missed events | every poll cycle | 10-min resync |
| Recovery after crash | next poll detects persisting drift | AddFunc re-fires for drifted resources on restart |
| Code delta vs A | baseline | ~+30 lines (informer setup) |

---

## New Files

Identical set to Approach A. Only the internal implementation of `cmd/drift-watcher/main.go` differs.

```
backend/temporal-worker/cmd/drift-watcher/main.go    NEW (informer-based)
backend/temporal-worker/internal/workflows/drift_approval.go  NEW (shared)
backend/temporal-worker/internal/activities/drift.go          NEW (shared)
backend/temporal-worker/cmd/worker/main.go                    MODIFY
k8s/drift-watcher/deployment.yaml                             NEW
k8s/drift-watcher/configmap.yaml                              NEW
k8s/temporal-worker/serviceaccount.yaml                       MODIFY
crossplane/xrd/* (4 files)                                    MODIFY
crossplane/composition/* (4 files)                            MODIFY
```

---

## Failure Modes

| Failure | Effect | Recovery |
|---------|--------|----------|
| Watcher pod crashes | Watch streams torn down | On restart, AddFunc fires for any still-drifted resources — no events permanently lost |
| K8s API server briefly unavailable | Watch stream disconnects | client-go auto-reconnects with exponential backoff |
| Watch stream falls behind ("too old resource version") | K8s closes stream | Informer detects and does a full re-list automatically |
| Temporal unavailable | `ExecuteWorkflow` fails, logged | Next XR update or resync fires a retry |

---

## Why This is the Production-Grade Version of Approach A

1. **No blind window on startup** — AddFunc fires for already-drifted resources on cache sync.
2. **Lower K8s API load** — one persistent TCP connection per GVR vs repeated list calls.
3. **Sub-second reaction time** — informer push vs up to 30s poll lag.
4. **Guaranteed delivery via resync** — 10-min resync catches anything missed during failures.

For the POC, Approach A is faster to implement. For production, migrate to Approach C.

---

## Pros and Cons

### Pros
- **Event-driven with safety net** — sub-second reaction from the watch stream, plus 10-min resync catches any missed events
- **No blind window on startup** — `AddFunc` fires for already-drifted resources when the informer cache first syncs; pod restart never loses events
- **Lower K8s API load than A** — 4 persistent watch streams (idle) vs repeated list calls every 30s
- **Go-only, no alpha dependencies** — same codebase as A, production-ready
- **Most resilient of the external binary approaches** — missed events recovered by resync; disconnected streams reconnected by client-go automatically

### Cons
- **More complex than A** — informer factory setup adds ~30 lines vs a simple loop; slightly harder to debug
- **New K8s Deployment** — same operational cost as A
- **Pod restart required for new GVRs** — informers are created at startup; adding a new resource type needs a pod restart
- **Informer event buffer can overflow** — under extremely high change rates the K8s API server may close the stream; mitigated by resync, but possible in theory
