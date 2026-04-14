---
title: "Approach A — Polling Watcher"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Approach A — Go Polling Watcher

**Branch:** `feature/drift-poc-approach-a-watcher`
**Trigger mechanism:** Periodic K8s API list-poll on all monitored MRs

---

## How It Works

A new Go binary (`drift-watcher`) runs as a Kubernetes Deployment. On each poll cycle
(default every 30s), it lists all MRs with `platform.io/drift-protection: "true"` across
all configured MR types, diffs `spec.forProvider` against `status.atProvider`, and fires
`DriftApprovalWorkflow` for any that have drifted. The workflow is keyed on the XR name,
resolved from the MR's `ownerReferences`.

```
Every 30s:
  for each configured MR GVR:
    list MRs with label platform.io/drift-protection=true
    for each MR:
      if forProvider != atProvider (drift detected):
        resolve ownerReferences → XR name
        ExecuteWorkflow(DriftApprovalWorkflow, mrFields + xrFields)
        → Temporal deduplicates via workflow ID (keyed on XR)
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Kubernetes                                                  │
│                                                             │
│  DatabaseInstance / Instance / Cluster / Bucket            │
│  (MRs with label platform.io/drift-protection=true)        │
│    └── spec.forProvider.settings.tier: db-n1-standard-2    │
│        status.atProvider.settings.tier: db-n1-standard-4   │
│                  │  diff detected                           │
│                  │  ownerRef → XDatabase/my-postgres        │
│                  │                                          │
│  drift-watcher (polls every 30s)                            │
│    for each MR GVR in config:                               │
│      list MRs → diff forProvider/atProvider → fire if drifted│
└──────────────────────────┬──────────────────────────────────┘
                           │  Temporal Go SDK
                           ▼
              DriftApprovalWorkflow
              (see Shared Design)
```

---

## Multi-Resource Support

The watcher reads its target GVRs from a ConfigMap. GVRs are **MR types**, not XR types —
the drift signal (`forProvider` vs `atProvider`) lives on the MR.

```yaml
# ConfigMap: drift-watcher-config
MR_GVRS: |
  sql.gcp.upbound.io/v1beta2/databaseinstances
  compute.gcp.upbound.io/v1beta2/instances
  container.gcp.upbound.io/v1beta2/clusters
  container.gcp.upbound.io/v1beta2/nodepools
  storage.gcp.upbound.io/v1beta2/buckets
```

For multi-MR XRs (e.g. GKE: `clusters` + `nodepools`), both MRs resolve to the same XR
via `ownerReferences`. Temporal deduplicates via workflow ID — only one approval workflow
runs per XR.

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
    gvrs  := parseGVRs(getenv("MR_GVRS", ""))

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
            drifted, detail := isDrifted(&item)
            if drifted {
                fireWorkflow(ctx, tc, buildDriftInput(&item, gvr, detail))
            }
        }
    }
}

// isDrifted compares spec.forProvider against status.atProvider.
// Returns true and a human-readable detail string when any forProvider field
// differs from the corresponding atProvider field.
// Fields present in atProvider but absent in forProvider are ignored (computed fields).
func isDrifted(obj *unstructured.Unstructured) (bool, string) {
    forProvider, _, _ := unstructured.NestedMap(obj.Object, "spec", "forProvider")
    atProvider, _, _  := unstructured.NestedMap(obj.Object, "status", "atProvider")
    if len(atProvider) == 0 {
        return false, "" // Observe() has not run yet
    }
    diffs := diffMaps("", forProvider, atProvider)
    if len(diffs) == 0 {
        return false, ""
    }
    return true, strings.Join(diffs, "; ")
}

// diffMaps recursively compares keys in desired against observed.
func diffMaps(prefix string, desired, observed map[string]interface{}) []string {
    var out []string
    for k, dv := range desired {
        path := k
        if prefix != "" {
            path = prefix + "." + k
        }
        ov := observed[k]
        if dMap, ok := dv.(map[string]interface{}); ok {
            if oMap, ok := ov.(map[string]interface{}); ok {
                out = append(out, diffMaps(path, dMap, oMap)...)
            } else {
                out = append(out, fmt.Sprintf("%s: differs", path))
            }
        } else if fmt.Sprintf("%v", dv) != fmt.Sprintf("%v", ov) {
            out = append(out, fmt.Sprintf("%s: %v → %v", path, dv, ov))
        }
    }
    return out
}

// buildDriftInput populates both MR and XR fields.
// XR fields are resolved from the MR's ownerReferences (controller owner).
func buildDriftInput(mr *unstructured.Unstructured, mrGVR schema.GroupVersionResource, detail string) DriftApprovalInput {
    xrName, xrKind, xrAPIVersion := resolveControllerOwner(mr)
    xrGroup, xrVersion := splitAPIVersion(xrAPIVersion)
    return DriftApprovalInput{
        MRGroup:     mrGVR.Group,
        MRVersion:   mrGVR.Version,
        MRResource:  mrGVR.Resource,
        MRName:      mr.GetName(),
        MRNamespace: mr.GetNamespace(),
        XRGroup:     xrGroup,
        XRVersion:   xrVersion,
        XRResource:  pluralFromKind(xrKind),
        XRKind:      xrKind,
        XRName:      xrName,
        XRNamespace: mr.GetNamespace(),
        DetectedAt:  time.Now().UTC().Format(time.RFC3339),
        DriftDetail: detail,
    }
}

func resolveControllerOwner(obj *unstructured.Unstructured) (name, kind, apiVersion string) {
    for _, ref := range obj.GetOwnerReferences() {
        if ref.Controller != nil && *ref.Controller {
            return ref.Name, ref.Kind, ref.APIVersion
        }
    }
    return "", "", ""
}

func fireWorkflow(ctx context.Context, tc client.Client, in DriftApprovalInput) error {
    workflowID := fmt.Sprintf("drift-approval-%s-%s-%s",
        in.XRNamespace, strings.ToLower(in.XRKind), in.XRName)
    _, err := tc.ExecuteWorkflow(ctx, client.StartWorkflowOptions{
        ID: workflowID, TaskQueue: "db-provisioning",
    }, workflows.DriftApprovalWorkflow, in)
    var alreadyStarted *serviceerror.WorkflowExecutionAlreadyStartedError
    if errors.As(err, &alreadyStarted) {
        return nil // dedup — workflow already running for this XR
    }
    return err
}
```

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `TEMPORAL_ADDRESS` | `localhost:7233` | Temporal frontend |
| `TEMPORAL_TASK_QUEUE` | `db-provisioning` | Task queue |
| `DRIFT_POLL_INTERVAL` | `30s` | Poll interval |
| `MR_GVRS` | (required) | Newline-separated `group/version/resource` list of MR types |

---

## Failure Modes

| Failure | Effect | Recovery |
|---|---|---|
| drift-watcher pod crashes | No new workflows fired during downtime | K8s restarts pod; next poll detects any persisting drift |
| Temporal unavailable on poll | `ExecuteWorkflow` fails, logged | Next poll cycle retries |
| `FlipManagementPolicyActivity` R3 fails | MR left in full management | CRITICAL log; Crossplane reconciles; next drift cycle restarts |
| Multiple replicas (if scaled) | Both try to fire same workflow | Temporal dedup via workflow ID — one succeeds |

---

## Pros and Cons

### Pros
- **Simplest implementation** — a single polling loop; easy to read, understand, and debug
- **No alpha dependencies** — works with any Crossplane version, no special flags or CRD types
- **Easy crash recovery** — pod restart picks up any still-drifted resources on the next poll cycle
- **Go-only** — no new languages or runtimes in the codebase

### Cons
- **Up to 30s detection lag** on top of Crossplane's own ~10 min poll interval (configurable via `--poll-interval`)
- **New K8s Deployment** — one more component to operate, monitor, and keep healthy
- **K8s API load grows linearly** with resource count × GVR count (low now, worth watching at scale)
- **Pod restart required for new GVRs** — `MR_GVRS` is read from env var at startup; adding a resource type needs a pod restart
- **Narrow loss window** — if the pod crashes between detecting drift and calling `ExecuteWorkflow`, that specific event is missed until the next poll cycle

---

## Limitations

- **Polling latency:** up to `DRIFT_POLL_INTERVAL` (30s) added on top of Crossplane's atProvider refresh cycle.
- **Stateless:** a crash window exists between detecting drift and calling `ExecuteWorkflow`.
- **K8s API load:** grows linearly with resource count. At 50 resources × 5 MR GVRs ≈ 10 list calls/min. Low.
