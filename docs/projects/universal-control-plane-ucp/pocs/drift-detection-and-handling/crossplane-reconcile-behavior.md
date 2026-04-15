---
title: "Crossplane Reconcile & Polling Behavior"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Crossplane Reconcile & Polling Behavior

This document clarifies how Crossplane's internal reconcile loop works, how it detects
drift, and why our custom drift detection adds value on top of it.

---

## What Crossplane Does vs What We Add

Crossplane already detects drift — it polls the actual GCP resource and compares it
against the desired state. But it does this **silently**:

- With `managementPolicies: ["*"]` (default): detects drift, auto-heals by calling
  Update/Create/Delete on GCP. No notification to anyone.
- With `managementPolicies: ["Observe"]`: detects drift, does nothing. Still no
  notification.

**Our custom drift detection adds the notification layer** — the part Crossplane has
no built-in support for. When drift is detected we can log it, signal a Temporal
workflow, page an on-call engineer, etc.

---

## Poll and Sync Intervals

> **Source:** The Crossplane official documentation does not document `--poll` or `--sync`
> default values. The values below were obtained by running `--help` directly on the
> provider binary inside the running cluster pod. See References section.

Verified directly from the provider binary on the running cluster:

```bash
# 1. Find the actual deployment name
kubectl get deployments -n crossplane-system | grep provider-gcp-compute

# 2. Run --help (2>&1 required — help output goes to stderr)
$ kubectl exec -n crossplane-system deployment/provider-gcp-compute-<actual-hash> \
    -- /usr/local/bin/provider --help 2>&1

--poll=10m    Poll interval controls how often an individual resource
              should be checked for drift. Default: 10 minutes.

--sync=1h     Sync interval controls how often all resources will be
              double checked for drift. Default: 1 hour.
```

| Flag | Default | Scope | Description |
|------|---------|-------|-------------|
| `--poll` | `10m` | Per resource | How often Observe() is called for a single managed resource |
| `--sync` | `1h` | All resources | Full re-list and re-sync of all managed resources — safety net for missed events |
| `--max-reconcile-rate` | `100/s` | Global | Max rate at which resources can be checked across the whole provider |

> **Note:** The installed provider is Terraform-based (`Terraform based Crossplane provider
> for GCP`). The flags above are from the actual running binary and reflect the Upbound
> official provider-gcp behavior.

---

## How the Provider Wakes Up to Reconcile

There are two triggers. Only one is time-based:

### 1. Kubernetes Watch Event (immediate)

Crossplane uses controller-runtime's watch mechanism. The provider keeps a long-lived
watch connection to the Kubernetes API server. When anything writes to a managed resource
object in Kubernetes, the API server pushes a notification and reconciliation starts
immediately — no waiting for the poll timer.

**What triggers a watch event:**
- User edits `spec.forProvider`
- User changes an annotation on the K8s object
- Crossplane itself writes to `status` (can retrigger the loop)
- Forcing a reconcile via: `kubectl annotate <resource> crossplane.io/paused=false --overwrite`

**What does NOT trigger a watch event:**
- A change made directly in GCP (e.g. adding a label in the GCP console)
- GCP-side changes are invisible to Kubernetes — they are only discovered when the poll
  timer fires and Observe() is called

### 2. Poll Timer (every 10 minutes)

Even without any Kubernetes change, the provider re-runs Observe() on every managed
resource on a fixed interval (`--poll=10m`). This is how out-of-band GCP changes are
eventually discovered and reflected in `status.atProvider`.

---

## Effect of `managementPolicies` on Poll Frequency

**None.** `managementPolicies` controls what operations the provider performs after
Observe() runs — it does not affect how often Observe() is called.

| managementPolicies | Observe() frequency | Reconcile action |
|--------------------|---------------------|-----------------|
| `["*"]` (default) | Every 10 min (poll) or on K8s event | Creates, updates, deletes on GCP to match desired state |
| `["Observe"]` | Every 10 min (poll) or on K8s event | Reads only — no write to GCP |
| `["ObserveAndDelete"]` | Every 10 min (poll) or on K8s event | Reads + deletes — no create/update |

The only way to change poll frequency is the `--poll` flag on the provider deployment,
configured via `DeploymentRuntimeConfig`:

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: provider-gcp-runtime
spec:
  deploymentTemplate:
    spec:
      template:
        spec:
          containers:
            - name: package-runtime
              args:
                - --poll=30s   # caution: high API call volume at scale
```

> **Warning:** Lowering `--poll` significantly (e.g. to 30s) multiplies GCP API calls
> by the number of managed resources. With 100 resources, 30s poll = 200 calls/minute
> against GCP APIs. Check GCP API quota limits before changing this.

---

## Two-Step Chain for Out-of-Band GCP Changes

When a change is made directly in GCP (outside of Crossplane), the update flows through
two separate loops before our drift watcher sees it:

```
Step 1 — Crossplane Observe() (every 10 min by default)
  GCP resource changes (e.g. label added in console)
        ↓
  provider-gcp calls GCP API (Observe)
        ↓
  writes updated state to status.atProvider in Kubernetes

Step 2 — Our drift watcher (every 30s)
  reads status.atProvider from Kubernetes API
        ↓
  compares against spec.forProvider
        ↓
  logs DRIFT_DETECTED / fires Temporal workflow
```

Our watcher only reads what Crossplane has already written to `atProvider`. If Step 1
has not run yet since the GCP change, Step 2 sees stale data. The worst-case detection
lag for out-of-band GCP changes is therefore:

```
worst-case lag = --poll interval (default 10 min) + our poll interval (30s)
               ≈ 10 min 30s
```

---

## Forcing an Immediate Reconcile

To bypass the poll timer and trigger Observe() immediately:

```bash
kubectl annotate <gvr> <resource-name> \
  crossplane.io/paused=false --overwrite
```

Example:
```bash
kubectl annotate instances.compute.gcp.upbound.io ari-test-compute-1-drift-protection \
  crossplane.io/paused=false --overwrite
```

---

## References

| Source | Notes |
|--------|-------|
| **Provider binary** — `kubectl exec -n crossplane-system deployment/provider-gcp-compute-<actual-hash> -- /usr/local/bin/provider --help 2>&1` (get hash via `kubectl get deployments -n crossplane-system \| grep provider-gcp-compute`) | **Primary and only source for `--poll=10m`, `--sync=1h`, and `--max-reconcile-rate=100` defaults.** The Crossplane official docs do not document these values anywhere. Verified on the running cluster. Note: `2>&1` is required — `--help` writes to stderr. |
| [Crossplane v2.2 — Managed Resources](https://docs.crossplane.io/v2.2/managed-resources/managed-resources/) | Official docs on managed resource lifecycle and `managementPolicies`. Does not cover poll/sync intervals. |
| [Crossplane v2.2 — Providers & DeploymentRuntimeConfig](https://docs.crossplane.io/v2.2/packages/providers/) | Official docs on configuring provider runtime via `DeploymentRuntimeConfig`, including passing args to the provider pod. |
| [crossplane/crossplane GitHub](https://github.com/crossplane/crossplane) | Source for core reconcile loop and controller-runtime usage |
| [upbound/provider-gcp GitHub](https://github.com/upbound/provider-gcp) | Source for the Terraform-based GCP provider |
| [kubernetes-sigs/controller-runtime GitHub](https://github.com/kubernetes-sigs/controller-runtime) | How Kubernetes watch events work under the hood |
