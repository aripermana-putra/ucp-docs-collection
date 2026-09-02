---
title: "Cloud Monitoring (GMP) — GKE Sandbox PoC — PoC Report"
space: UCP
parent_page_id: "../cloud-monitoring.md"
---

# Cloud Monitoring (GMP) — GKE Sandbox PoC — PoC Report

Human-first verdict. See [design.md](design.md) for scope/hypothesis and
[implementation.md](implementation.md) for full execution detail.

## Research question answered

This PoC answers the question in [design.md](design.md#research-question): whether Option A
delivers GKE resource metrics, a custom application metric, and Cloud SQL infra metrics at the
near-zero operational overhead the parent research's claims imply, measured hands-on rather than
assumed. It does not answer the Option A vs Option B comparison — that comparison is written
into the [parent research document](../../research/system-resource-monitoring.md) once both
this PoC and the [MonaaS OTel Collector PoC](../monaas-otel-collector.md) have run.

## Verdict

All five phases in [design.md](design.md#approach) passed. GKE resource metrics and Cloud SQL
infra metrics were already present in Cloud Monitoring before this PoC touched the cluster — no
manifest, component, or IAM grant was needed for either. The custom application metric required
exactly one manifest: a `PodMonitoring` CRD, which GMP's already-running collector picked up
immediately with no scrape or permission errors.

**Cloud SQL metrics carry per-instance identity natively; Option B's do not.** Querying Cloud
Monitoring directly for `test-db-metrics`'s CPU utilization returns a `database_id` resource
label identifying which instance produced it. The [MonaaS OTel Collector
PoC](../monaas-otel-collector/poc-report.md)'s `googlecloudmonitoringreceiver` output for the
same metric carries no such label — a real limitation for a production topology with 2+ Cloud
SQL instances (see that PoC's [design.md open
questions](../monaas-otel-collector/design.md#open-questions)). Option A does not have this
problem: the resource label survives because the query goes directly against Cloud Monitoring's
own metric namespace rather than being re-exported through a Prometheus-format receiver.

## Manifest and component count

| | Count |
|---|---|
| Manifests applied | 1 (`01-podmonitoring.yaml`) |
| Distinct Kubernetes objects from the applied manifest | 1 (`PodMonitoring/sample-app`) |
| Distinct self-managed software component types | 0 |

No component is deployed or maintained by the tenant under Option A for any of the three metric
categories in this PoC's scope. The `PodMonitoring` CRD is a config object, not a component —
there is no binary, controller, or operator introduced by applying it. GMP's own collector and
`gmp-operator` are Google-managed and were already running before this PoC started; Cloud SQL's
infra-metric emission is automatic and required no configuration at all.

### What requires ongoing maintenance

GKE resource metrics and Cloud SQL infra metrics require zero ongoing action under Option A —
both are automatic per node, pod, and Cloud SQL instance, with nothing to configure as new
infrastructure is added.

Custom application metrics are not zero-maintenance: every new app that needs a custom metric
scraped requires its own `PodMonitoring`/`ClusterPodMonitoring` CRD. This is a real, recurring
action per app — the same shape as adding a `ServiceMonitor` in Option C. It is smaller and more
isolated than Option B's OTel Collector variant, where every new scrape target means editing a
shared Collector CR's `scrape_configs` (see [MonaaS OTel Collector PoC — Component
detail](../monaas-otel-collector/poc-report.md#component-detail-type-and-setup-vs-ongoing-maintenance)),
but "a small CRD per app" is still an ongoing task, not nothing.

## What this PoC did not prove

- Native Cloud Monitoring Dashboards vs Grafana as the UI/alerting layer — out of scope per
  [design.md](design.md#scope); this PoC only verified data reaches Cloud Monitoring and is
  queryable, not which UI a tenant should use.
- Alert-policy configuration — out of scope per [design.md](design.md#scope).
- Production-scale traffic or query load — sandbox traffic is not representative.
- Cost at production scale.
- The Option A vs Option B comparison itself.

## Evidence: PromQL query results

Queried via the GMP Prometheus-compatible query API (`.../location/global/prometheus/api/v1/query`),
authenticated with `gcloud auth print-access-token`. Full detail in
[implementation.md](implementation.md#verification).

**GKE resource metric — already present, zero setup:**
```json
{"metric":{"__name__":"kubernetes_io:node_cpu_core_usage_time","node_name":"gke-ucp-agent-cluster-spot-pool-76eae683-qlt9", ...},"value":[1788314816.797,"3164.681"]}
```

**Custom app metric — via the single `PodMonitoring` CRD:**
```json
{"metric":{"__name__":"http_requests_total","code":"200","container":"prometheus-example-app","namespace":"default","pod":"prometheus-example-app-849cf95954-5nxhh", ...},"value":[1788315051.194,"10"]}
```

**Cloud SQL infra metric — already present, zero setup, resource-labeled:**
```json
{"metric":{"__name__":"cloudsql_googleapis_com:database_cpu_utilization","database_id":"sub-gcp-ucp-clsd-sandbox:test-db-metrics","monitored_resource":"cloudsql_database", ...},"value":[1788314950.193,"0.09330182984680893"]}
```

## Recommendation

No change to the parent research's Recommendation (Option A). This PoC confirms Option A's
zero-collector claims for all three metric categories hands-on, and adds one new finding not
previously documented: Cloud SQL metric identity survives per-instance under Option A but is
lost under Option B's `googlecloudmonitoringreceiver`. The Option A vs Option B comparison
itself is written into the [parent research document](../../research/system-resource-monitoring.md).
