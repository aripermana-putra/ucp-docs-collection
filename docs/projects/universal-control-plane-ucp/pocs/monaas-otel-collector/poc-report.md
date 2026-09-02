---
title: "MonaaS OTel Collector Variant — PoC Report"
space: UCP
parent_page_id: "../monaas-otel-collector.md"
---

# MonaaS OTel Collector Variant — PoC Report

Human-first verdict. See [design.md](design.md) for scope/hypothesis and
[implementation.md](implementation.md) for full execution detail.

## Research question answered

This PoC answers the question in [design.md](design.md#research-question): whether Option B's
OTel Collector variant delivers GKE resource metrics, a custom application metric, and Cloud SQL
infra metrics with the operational overhead its component-count estimate implies, measured
hands-on rather than assumed. It does not answer the Option A vs Option B comparison — that
comparison is written into the [parent research document](../../research/system-resource-monitoring.md)
once both this PoC and the [Cloud Monitoring PoC](../cloud-monitoring.md) have run.

## Verdict

All seven phases in [design.md](design.md#approach) passed: `hostmetricsreceiver`,
`kubeletstatsreceiver`, `k8sclusterreceiver`, `prometheusreceiver`, and
`googlecloudmonitoringreceiver` all produced queryable metrics in an in-cluster Mimir,
remote-written from two `OpenTelemetryCollector` custom resources. Two Collector-internal
issues surfaced during hands-on execution: one image-architecture fix on the sample workload
(see below), and a missing `resourcedetection` processor on the DaemonSet Collector — without
it, `hostmetricsreceiver` carries no per-node identity, so metrics from every node collide into
a single unlabeled series in Mimir regardless of node or pool.

**Workload Identity is not enabled on the sandbox cluster.** `googlecloudmonitoringreceiver`'s
GCP authentication was granted via `roles/monitoring.viewer` on the node pools' default Compute
Engine service account instead — a project-level IAM change, not a Kubernetes object, that
grants read-only Cloud Monitoring access to every pod on every node pool rather than only the
Collector. A cluster with Workload Identity enabled would scope this to the Collector's own
identity instead; that scoping was not exercised here.

**The parent research's component-count estimate for this variant undercounts by one.** The
[Components table](../../research/system-resource-monitoring.md#components-to-configure-and-manage)
estimates "~2 (Operator, Collector)" self-managed components for the full scope. Hands-on
execution required **3**: `cert-manager`, `opentelemetry-operator`, and the OTel Collector
binary (deployed as two separate workloads — a DaemonSet and a Deployment — but one binary
type). `cert-manager` was already flagged as a risk in design.md's Risks table, but the
Components table itself was not updated to reflect it as a distinct component until this PoC
ran.

## Manifest and component count

| | Count |
|---|---|
| Manifests applied | 6 (`cert-manager.yaml`, `opentelemetry-operator.yaml` upstream; `01-mimir.yaml`, `02-otel-daemonset.yaml`, `03-sample-app.yaml`, `04-otel-deployment.yaml` authored) |
| Distinct Kubernetes objects from the 6 applied manifests | 81 (47 from `cert-manager`, 20 from `opentelemetry-operator`, 14 authored) — plus additional objects generated at runtime by the operators/reconciler, see [implementation.md](implementation.md#object-inventory-live-cluster-state) |
| Distinct self-managed software component types | 3 (`cert-manager`, `opentelemetry-operator`, OTel Collector) |
| Collector workloads | 2 (DaemonSet, cluster-wide across all node pools; Deployment on `spot-pool`) |

### Component detail: type, and setup vs. ongoing maintenance

Three components count toward this variant's tenant-side operational burden. Mimir and the
sample workload are excluded from that count — Mimir stands in for MonaaS's Cortex backend,
which is entirely OneCloud-managed in the real Option B (not deployed by the tenant at all);
the sample workload is the app being observed, not part of the monitoring stack itself.

| Component | Type | Workload shape | Setup | Ongoing maintenance |
|---|---|---|---|---|
| `cert-manager` | Certificate-management operator (prerequisite) | 3 Deployments (controller, webhook, cainjector), 47 objects total from the upstream manifest | One-time install per cluster | Version upgrades tied to the K8s API version support window; certificate issuance/rotation for its own webhook is automatic once installed — no manual cert handling. Not touched by adding new receivers or new app metrics. |
| `opentelemetry-operator` | Kubernetes controller managing the `OpenTelemetryCollector` CRD lifecycle | 1 Deployment, 20 objects total from the upstream manifest | One-time install per cluster | Version upgrades — tied to which Collector contrib image versions it validates against; depends on `cert-manager` staying healthy for its own webhook certs. Not touched by adding new receivers or new app metrics — those are Collector CR edits, not operator changes. |
| OTel Collector (`otel/opentelemetry-collector-contrib`) | Collector/scraper binary — one binary type, two deployment shapes | DaemonSet (1 pod/node, cluster-wide across every node pool) for `hostmetrics`+`kubeletstats`; Deployment (1 pod on `spot-pool`) for `k8scluster`+`prometheus`+`googlecloudmonitoring` receiver + `prometheus_remote_write` exporter | Initial receiver/exporter config authored once per Collector CR — cluster-wide node coverage also requires a `resourcedetection` processor and `resource_to_telemetry_conversion: enabled: true` on the exporter, without which every DaemonSet pod's metrics collide into one unlabeled series regardless of node; `googlecloudmonitoringreceiver` additionally requires a GCP IAM grant (`roles/monitoring.viewer`), which is a config/IAM change, not a new deployed component | **This is the component with recurring, hands-on maintenance:** every new app-metric target means editing the Deployment Collector's `prometheus` receiver `scrape_configs` (centralized in one CR instead of a per-app CRD like `PodMonitoring`/`ServiceMonitor`); new GCP-emitted metric types (e.g. a second Cloud SQL instance, or a different managed service) mean editing `metrics_list` on the same CR; new K8s object types to observe need RBAC additions to the `ClusterRole`; image version bumps for receiver features/security patches; resource requests/limits need tuning as node count or pod density grows, since the DaemonSet runs on every node in every pool alongside real workloads |

By contrast, Option A's `PodMonitoring` CRD and Option C's `ServiceMonitor`/`PodMonitor` CRDs
push new-target onboarding to a small, per-app CRD rather than a shared Collector CR — this
PoC did not measure that difference quantitatively, but it is the same "one central scrape
config vs. per-app CRD" shape noted in the
[Components table](../../research/system-resource-monitoring.md#components-to-configure-and-manage).

## Wall-clock time

**~19 minutes** from `spot-pool` resize to the original two metric categories
(`hostmetrics`/`kubeletstats`/`k8scluster`/`prometheus` receivers) confirmed queryable in Mimir
(derived from Kubernetes object timestamps — see
[implementation.md](implementation.md#wall-clock-time)). This excludes prior design/planning
time and the sandbox cluster access setup (gcloud/kubectl auth), which is one-time per operator
and not specific to this variant. Cloud SQL metrics via `googlecloudmonitoringreceiver` are not
included in this figure — see [implementation.md](implementation.md#sandbox-cluster) for the
IAM setup step they require.

## What this PoC did not prove

- Production-scale traffic or query load against Mimir — sandbox traffic is not representative
  (out of scope per [design.md](design.md#scope)).
- Real MonaaS onboarding (ACL, Bearer token, `auth-cloud-api` IP allowlisting) — Mimir has no
  auth step to replicate, so the onboarding overhead documented elsewhere in the parent research
  is not re-measured here.
- Cost at production scale.
- The Option A vs Option B comparison itself.
- Per-instance Cloud SQL metric identity at production scale — only one Cloud SQL instance
  (`test-db-metrics`) exists in the sandbox project, so the collision itself was not exercised
  directly, but the underlying limitation is confirmed: `googlecloudmonitoringreceiver` emits no
  resource label identifying which instance a metric came from (see
  [implementation.md](implementation.md#verification) for the same failure mode already
  confirmed on the DaemonSet Collector). A production topology with 2+ Cloud SQL instances
  (platform DB and Temporal DB, each active/standby) would see same-named metrics from
  different instances collide into one series. This is an open question requiring further
  exploration — see [design.md](design.md#open-questions).
- Workload Identity-scoped GCP authentication — the sandbox cluster does not have Workload
  Identity enabled, so this PoC used a broader, project-level IAM grant instead; a cluster with
  Workload Identity enabled would scope this to the Collector's own identity.

## Evidence: PromQL query results

Queried via `kubectl -n monitoring port-forward svc/mimir 8080:8080` and
`curl http://localhost:8080/prometheus/api/v1/query?query=...`.

**Resource metric — `hostmetricsreceiver` (DaemonSet Collector, cluster-wide):** one distinct
series per node, spanning both `system-pool` and `spot-pool`, confirming cluster-wide collection
across every node pool:
```json
{"status":"success","data":{"resultType":"vector","result":[
  {"metric":{"__name__":"system_cpu_logical_count","host_name":"gke-ucp-agent-cluster-system-pool-84a6ae3f-osr8", ...},"value":[1788253232.475,"2"]},
  {"metric":{"__name__":"system_cpu_logical_count","host_name":"gke-ucp-agent-cluster-spot-pool-76eae683-qlt9", ...},"value":[1788253232.475,"2"]},
  {"metric":{"__name__":"system_cpu_logical_count","host_name":"gke-ucp-agent-cluster-system-pool-e9664b53-bva1", ...},"value":[1788253232.475,"2"]},
  {"metric":{"__name__":"system_cpu_logical_count","host_name":"gke-ucp-agent-cluster-system-pool-40e9e664-j33y", ...},"value":[1788253232.475,"2"]}
]}}
```

**Container/pod resource metrics — `kubeletstatsreceiver`** (metric names present in Mimir):
```
k8s_pod_cpu_usage, k8s_pod_memory_usage_bytes, k8s_pod_memory_working_set_bytes,
k8s_pod_filesystem_usage_bytes, k8s_pod_network_io_bytes_total,
k8s_node_cpu_usage, k8s_node_memory_usage_bytes, k8s_node_filesystem_available_bytes,
k8s_node_condition_ready, container_cpu_usage, container_cpu_time_seconds_total
```

**K8s object-state metrics — `k8sclusterreceiver`** (metric names present in Mimir):
```
k8s_deployment_available, k8s_deployment_desired,
k8s_replicaset_available, k8s_replicaset_desired,
k8s_namespace_phase
```

**Custom app metric — `prometheusreceiver` scraping the sample workload:**
```json
{"status":"success","data":{"resultType":"vector","result":[
  {"metric":{"__name__":"http_requests_total","code":"200","method":"get",
    "instance":"prometheus-example-app.default.svc.cluster.local:8080","job":"sample-app"},
   "value":[1788248307.577,"1"]}
]}}
```

**Scrape health:**
```json
{"status":"success","data":{"resultType":"vector","result":[
  {"metric":{"__name__":"up","instance":"prometheus-example-app.default.svc.cluster.local:8080",
    "job":"sample-app"},"value":[1788248307.319,"1"]}
]}}
```

**Cloud SQL infra metrics — `googlecloudmonitoringreceiver`** (Deployment Collector, polling
the GCP Cloud Monitoring API for `test-db-metrics`):
```json
{"status":"success","data":{"resultType":"vector","result":[
  {"metric":{"__name__":"cloudsql_googleapis_com_database_cpu_utilization"},
   "value":[1788311509.669,"0.09761273904425925"]}
]}}
{"status":"success","data":{"resultType":"vector","result":[
  {"metric":{"__name__":"cloudsql_googleapis_com_database_memory_utilization"},
   "value":[1788311531.415,"0.25464304015392103"]}
]}}
{"status":"success","data":{"resultType":"vector","result":[
  {"metric":{"__name__":"cloudsql_googleapis_com_database_disk_utilization"},
   "value":[1788311531.528,"0.009585496182907301"]}
]}}
```
No resource label identifies `test-db-metrics` on any of these three series — see [What this
PoC did not prove](#what-this-poc-did-not-prove).

## Recommendation

No change to the parent research's Recommendation (Option A). This PoC's role is to supply
Option B's measured overhead for the final Option A vs Option B comparison, which happens in the
[parent research document](../../research/system-resource-monitoring.md) once the
[Cloud Monitoring PoC](../cloud-monitoring.md) has also run.
