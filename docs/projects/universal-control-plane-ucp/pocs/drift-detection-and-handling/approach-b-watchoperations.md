---
title: "Approach B — WatchOperations"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Approach B — WatchOperations + function-python

**Branch:** `feature/drift-poc-approach-b-watchoperations`
**Trigger mechanism:** Crossplane-native event-driven via WatchOperation → Operation → function-python

---

## How It Works

A `WatchOperation` resource watches all XRs with `platform.io/drift-protection: "true"`.
When any watched XR changes, Crossplane creates an `Operation` that runs `function-python`.
The function checks if the XR is actually drifted, then calls the Temporal Python SDK to
start `DriftApprovalWorkflow`. No external binary needed.

```
XR condition changes (event-driven, Crossplane watch stream)
    └── WatchOperation triggers Operation creation
            └── function-python executes:
                  1. check isDrifted(watchedXR)
                  2. if drifted → temporalio.client.start_workflow(DriftApprovalWorkflow)
                  3. if not drifted → no-op, complete
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Kubernetes / Crossplane                                  │
│                                                          │
│  WatchOperation (one per XR type)                        │
│  └── watch.kind: XDatabase / XComputeInstance / etc.    │
│       watch.matchLabels: drift-protection=true          │
│         │                                               │
│         │ XR changes (event-driven)                     │
│         ▼                                               │
│     Operation created → function-python pod runs        │
│       └── checks Synced=False                           │
│       └── calls Temporal gRPC :7233                     │
└──────────────────────────┬──────────────────────────────┘
                           │  Temporal Python SDK (gRPC)
                           ▼
              DriftApprovalWorkflow
              (see Shared Design)
```

---

## Crossplane Features Required

| Feature | Status | Notes |
|---------|--------|-------|
| ManagementPolicies | GA (beta) | Already available |
| WatchOperations | **Alpha** | Requires `--enable-operations` flag |
| Operations | **Alpha** | Same flag |
| function-python | Alpha | Must be installed separately |

**Alpha flag required:** Add `--set "args={--enable-operations}"` to the Crossplane Helm install.

---

## Multi-Resource Support

One `WatchOperation` per XR type, all pointing to the same `function-drift-notifier`:

```
crossplane/watchoperations/
├── drift-watch-xdatabase.yaml
├── drift-watch-xcomputeinstance.yaml
├── drift-watch-xkubernetescluster.yaml
└── drift-watch-xobjectstorage.yaml
```

Adding a new resource type = copy one WatchOperation YAML and update `watch.kind`.

---

## WatchOperation Manifest

```yaml
apiVersion: ops.crossplane.io/v1alpha1
kind: WatchOperation
metadata:
  name: drift-watch-xdatabase
spec:
  watch:
    apiVersion: platform.example.io/v1alpha1
    kind: XDatabase
    matchLabels:
      platform.io/drift-protection: "true"
  concurrencyPolicy: Allow
  successfulHistoryLimit: 5
  failedHistoryLimit: 5
  operationTemplate:
    spec:
      pipeline:
      - step: detect-and-notify
        functionRef:
          name: function-drift-notifier
        input:
          apiVersion: drift.platform.io/v1alpha1
          kind: DriftNotifierConfig
          spec:
            temporalAddress: "temporal-frontend.temporal-system:7233"
            temporalTaskQueue: "db-provisioning"
            temporalNamespace: "default"
            xrGroup: "platform.example.io"
            xrVersion: "v1alpha1"
            xrResource: "xdatabases"
            xrKind: "XDatabase"
```

---

## Python Function (main.py)

```python
async def run(req, _) -> fnv1.RunFunctionResponse:
    rsp = response.to(req)
    watched = request.get_required_resource(req, "ops.crossplane.io/watched-resource")
    name      = watched["metadata"]["name"]
    namespace = watched["metadata"].get("namespace", "default")
    config    = req.input["spec"]

    if not is_drifted(watched):
        return rsp  # no-op on every non-drifted change

    xr_kind     = config["xrKind"]
    workflow_id = f"drift-approval-{namespace}-{xr_kind.lower()}-{name}"

    await start_drift_workflow(
        address=config["temporalAddress"],
        namespace=config["temporalNamespace"],
        task_queue=config["temporalTaskQueue"],
        workflow_id=workflow_id,
        input_payload={
            "xrGroup": config["xrGroup"], "xrVersion": config["xrVersion"],
            "xrResource": config["xrResource"], "xrKind": xr_kind,
            "xrName": name, "namespace": namespace,
            "detectedAt": datetime.now(timezone.utc).isoformat(),
            "driftReason": get_synced_reason(watched),
        },
    )
    return rsp

def is_drifted(obj: dict) -> bool:
    for c in obj.get("status", {}).get("conditions", []):
        if c.get("type") == "Synced" and c.get("status") == "False":
            return c.get("reason", "") == "ReconcileError"
    return False
```

---

## Deduplication

Two layers:
1. **`is_drifted()` gate** — exits if XR is not actually drifted. Prevents Temporal calls on every reconcile tick.
2. **Temporal workflow ID** — deterministic `drift-approval-<ns>-<kind>-<name>`. `AlreadyStarted` swallowed silently.

---

## New Files

```
k8s/deploy-crossplane.sh                              MODIFY (--enable-operations)
crossplane/providers/function-python.yaml             NEW
crossplane/watchoperations/*.yaml (4 files)           NEW
crossplane/functions/drift-notifier/main.py           NEW
crossplane/functions/drift-notifier/requirements.txt  NEW
crossplane/functions/drift-notifier/Dockerfile        NEW
backend/temporal-worker/internal/workflows/drift_approval.go  NEW (shared)
backend/temporal-worker/internal/activities/drift.go          NEW (shared)
backend/temporal-worker/cmd/worker/main.go                    MODIFY
k8s/temporal-worker/serviceaccount.yaml                       MODIFY (RBAC)
```

---

## Additional Setup Required

```bash
# 1. Re-install Crossplane with Operations enabled
helm upgrade crossplane crossplane-stable/crossplane \
  --set "args={--enable-operations}" -n crossplane-system

# 2. Build and push function package (requires Crossplane CLI)
# A Crossplane function is an .xpkg package, not a plain Docker image.
cd crossplane/functions/drift-notifier

# Step 1: build the runtime image
docker build . --platform=linux/amd64 --tag drift-notifier-runtime:v0.1.0

# Step 2: build the Crossplane package embedding the runtime
crossplane xpkg build \
  --package-root=package \
  --embed-runtime-image=drift-notifier-runtime:v0.1.0 \
  --package-file=drift-notifier.xpkg

# Step 3: push the Crossplane package
crossplane xpkg push \
  --package-files=drift-notifier.xpkg \
  <registry>/drift-notifier:v0.1.0

# 3. Verify Temporal gRPC reachable from crossplane-system
kubectl run test --rm -it --image=alpine -n crossplane-system \
  -- sh -c "nc -zv temporal-frontend.temporal-system 7233"
```

---

## Failure Modes

| Failure | Effect | Recovery |
|---------|--------|----------|
| function-python crashes | Operation marked Failed | Crossplane retries per `failedHistoryLimit` |
| Temporal gRPC unreachable | `start_workflow` raises, Operation fails | Next XR change creates new Operation |
| `--enable-operations` flag missing | WatchOperation CRD not registered | Re-install Crossplane with flag |
| Function image unavailable | ImagePullBackOff | Push image, Operation auto-retries |

---

## Pros and Cons

### Pros
- **No new K8s Deployment** — function runs as a short-lived Crossplane Operation pod (ephemeral, managed by Crossplane)
- **Event-driven** — sub-second reaction after Crossplane propagates `Synced=False` to the XR; no poll delay
- **Declarative** — WatchOperation YAML is self-describing; visible to anyone inspecting the cluster
- **No persistent process** — Crossplane manages the Operation lifecycle; no pod to keep alive

### Cons
- **Alpha dependency** — `--enable-operations` flag and WatchOperations CRD are alpha; may change or break before graduating; not production-ready
- **Crossplane reinstall required** — enabling the alpha flag means re-running `helm upgrade` on Crossplane; disruptive in shared clusters
- **Python in a Go codebase** — new runtime, new language, new dependency supply chain for the team to maintain
- **Image build required** — function image must be built and pushed before anything works; adds a CI/CD step absent from all other approaches
- **Higher per-type extension cost** — adding a new resource type requires a full new WatchOperation YAML file; A/C only need one ConfigMap line, D only needs a schedule update command

---

## Limitations

- **Alpha dependency** — WatchOperations and Operations are alpha. Not production-ready until graduation.
- **Python in a Go codebase** — adds a new language and runtime to maintain.
- **Higher per-type extension cost** — all approaches require something when adding a new resource type; B requires a full new WatchOperation YAML file per type, whereas A/C only need one ConfigMap line and D only needs a `temporal schedule update` command.
- **Package build pipeline required** — the function is a Crossplane `.xpkg` package, not a plain Docker image. Building it requires `docker build` + `crossplane xpkg build` + `crossplane xpkg push`, plus a `package/crossplane.yaml` metadata file. This CI/CD step is absent from all other approaches.
- **Detection floor is still ~1 min** — gated by Crossplane's own poll interval, same as all other approaches.
