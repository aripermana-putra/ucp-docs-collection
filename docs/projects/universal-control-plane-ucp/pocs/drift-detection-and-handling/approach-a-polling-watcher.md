---
title: "Approach A — Polling Watcher"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Approach A — Go Polling Watcher

**Branch:** `feature/drift-poc-approach-a-watcher`
**Trigger mechanism:** Periodic K8s API list-poll on all monitored XRs

---

## How It Works

A new Go binary (`drift-watcher`) runs as a Kubernetes Deployment. On each poll cycle
(default every 30s), it lists all XRs with `platform.io/drift-protection: "true"` across
all configured XR types, checks their `Synced` condition, and fires `DriftApprovalWorkflow`
for any that are drifted.

```
Every 30s:
  for each configured XR GVR:
    list XRs with label platform.io/drift-protection=true
    for each XR:
      if Synced=False (ReconcileError):
        ExecuteWorkflow(DriftApprovalWorkflow, ...)
        → Temporal deduplicates via workflow ID
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Kubernetes                                              │
│                                                         │
│  XDatabase / XComputeInstance / XKubernetesCluster /   │
│  XObjectStorage (any XR with drift-protection label)   │
│    └── status.conditions[Synced] = False               │
│                  │                                      │
│  drift-watcher (polls every 30s)                        │
│    for each XR GVR in config:                           │
│      list XRs → check conditions → fire if drifted     │
└──────────────────────────┬──────────────────────────────┘
                           │  Temporal Go SDK
                           ▼
              DriftApprovalWorkflow
              (see Shared Design)
```

---

## Multi-Resource Support

The watcher reads its target GVRs from a ConfigMap:

```yaml
# ConfigMap: drift-watcher-config
XR_GVRS: |
  platform.example.io/v1alpha1/xdatabases
  platform.example.io/v1alpha1/xcomputeinstances
  platform.example.io/v1alpha1/xkubernetesclusters
  platform.example.io/v1alpha1/xobjectstorages
```

Adding a new resource type = add one line to the ConfigMap. No code change, no redeployment.

---

## New Files

```
backend/temporal-worker/cmd/drift-watcher/main.go   NEW
backend/temporal-worker/internal/workflows/drift_approval.go   NEW (shared)
backend/temporal-worker/internal/activities/drift.go           NEW (shared)
backend/temporal-worker/cmd/worker/main.go                     MODIFY
k8s/drift-watcher/deployment.yaml                              NEW
k8s/drift-watcher/configmap.yaml                               NEW
k8s/temporal-worker/serviceaccount.yaml                        MODIFY (RBAC)
crossplane/xrd/* (4 files)                                     MODIFY
crossplane/composition/* (4 files)                             MODIFY
```

---

## Key Code: drift-watcher/main.go

```go
func main() {
    tc, _ := client.Dial(client.Options{HostPort: getenv("TEMPORAL_ADDRESS", "localhost:7233")})
    dc, _ := k8s.NewDynamicClient()
    gvrs  := parseGVRs(getenv("XR_GVRS", ""))

    for {
        pollCycle(context.Background(), dc, tc, gvrs)
        time.Sleep(parseDuration(getenv("DRIFT_POLL_INTERVAL", "30s")))
    }
}

func pollCycle(ctx context.Context, dc dynamic.Interface, tc client.Client, gvrs []schema.GroupVersionResource) {
    for _, gvr := range gvrs {
        list, _ := dc.Resource(gvr).Namespace("").List(ctx, metav1.ListOptions{
            LabelSelector: "platform.io/drift-protection=true",
        })
        for _, item := range list.Items {
            if isDrifted(&item) {
                fireWorkflow(ctx, tc, buildDriftInput(&item, gvr))
            }
        }
    }
}

func isDrifted(obj *unstructured.Unstructured) bool {
    conditions, _, _ := unstructured.NestedSlice(obj.Object, "status", "conditions")
    for _, c := range conditions {
        cond, _ := c.(map[string]interface{})
        if cond["type"] == "Synced" && cond["status"] == "False" {
            reason, _ := cond["reason"].(string)
            return reason == "ReconcileError"
        }
    }
    return false
}

func fireWorkflow(ctx context.Context, tc client.Client, in DriftApprovalInput) error {
    workflowID := fmt.Sprintf("drift-approval-%s-%s-%s",
        in.Namespace, strings.ToLower(in.XRKind), in.XRName)
    _, err := tc.ExecuteWorkflow(ctx, client.StartWorkflowOptions{
        ID: workflowID, TaskQueue: "db-provisioning",
    }, workflows.DriftApprovalWorkflow, in)
    if errors.As(err, &serviceerror.WorkflowExecutionAlreadyStartedError{}) {
        return nil // dedup success
    }
    return err
}
```

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `TEMPORAL_ADDRESS` | `localhost:7233` | Temporal frontend |
| `TEMPORAL_TASK_QUEUE` | `db-provisioning` | Task queue |
| `DRIFT_POLL_INTERVAL` | `30s` | Poll interval |
| `XR_GVRS` | (required) | Newline-separated `group/version/resource` list |

---

## Failure Modes

| Failure | Effect | Recovery |
|---------|--------|----------|
| drift-watcher pod crashes | No new workflows fired during downtime | K8s restarts pod; next poll detects any persisting drift |
| Temporal unavailable on poll | `ExecuteWorkflow` fails, logged | Next poll cycle retries |
| FlipManagementPolicyActivity R3 fails | Resource left in full management | CRITICAL log; Crossplane reconciles; next drift cycle restarts |
| Multiple replicas (if scaled) | Both try to fire same workflow | Temporal dedup via workflow ID — one succeeds |

---

## Pros and Cons

### Pros
- **Simplest implementation** — a single polling loop; easy to read, understand, and debug
- **No alpha dependencies** — works with any Crossplane version, no special flags or CRD types
- **Easy crash recovery** — pod restart picks up any still-drifted resources on the next poll cycle
- **Go-only** — no new languages or runtimes in the codebase

### Cons
- **Up to 30s detection lag** after Crossplane sets `Synced=False` (poll interval adds latency on top of Crossplane's ~1 min)
- **New K8s Deployment** — one more component to operate, monitor, and keep healthy
- **K8s API load grows linearly** with resource count × GVR count (low now, worth watching at scale)
- **Pod restart required for new GVRs** — `XR_GVRS` is read from env var at startup; adding a resource type needs a pod restart
- **Narrow loss window** — if the pod crashes between detecting drift and calling `ExecuteWorkflow`, that specific event is missed until the next poll cycle

---

## Limitations

- **Polling latency:** up to `DRIFT_POLL_INTERVAL` (30s) after Crossplane detects drift.
  Total end-to-end latency ≈ 60–90s.
- **Stateless:** a crash window exists between detecting drift and calling `ExecuteWorkflow`.
- **K8s API load:** grows linearly with resource count. At 50 resources × 4 GVRs ≈ 8 list calls/min. Low.
