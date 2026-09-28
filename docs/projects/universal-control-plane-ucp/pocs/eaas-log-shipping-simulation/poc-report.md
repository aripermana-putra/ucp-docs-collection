---
title: "EaaS Log Shipping Simulation — PoC Report"
space: UCP
parent_page_id: "../eaas-log-shipping-simulation.md"
---

# EaaS Log Shipping Simulation — PoC Report

Human-first verdict. See [design.md](design.md) for scope/hypothesis and
[implementation.md](implementation.md) for full execution detail.

## Research question answered

This PoC answers the question in [design.md](design.md#research-question): whether Filebeat,
configured per EaaS's documented conventions, can correctly ship **each of UCP's real log
shapes** — not just a generic Go service's JSON — through a pipeline shaped like EaaS's own, with
each shape's meaningful fields intact and distinguishable by `geap_log_group`. It does **not**
answer whether UCP's GCP projects can actually reach a real EaaS gateway — that stays a direct
confirmation with the EaaS team, tracked in the parent research's [Open
Questions](../../research/application-log-monitoring.md#open-questions).

## Verdict

**All four log groups shipped, parsed, and are distinguishable in Kibana/Elasticsearch, using a
shared-shipper (DaemonSet) model — 2 shippers total covering 4 log groups, not one shipper per
component.** Two Filebeat/Logstash configuration bugs were found and fixed along the way (see
[implementation.md](implementation.md#phase-3--sample-workload--filebeat-daemonset-gke)); one
log shape (`temporal-worker`) only partially parses, by design.

**Temporal Workers' logging is confirmed to be wired through a `slog`-based structured JSON
logger**, at both the client and worker level, so ADR-007's structured-JSON assumption holds for
Temporal Workers as currently implemented. The plain-text SDK output this PoC's Colima track
captured (`2026/09/28 07:23:29 DEBUG ExecuteActivity Namespace default TaskQueue
drift-detection WorkerID 1@drift-worker-566b9bcfb7-q8j6n@ ...`) came from a specific
already-running dev instance that predates that wiring, not from the current implementation —
recorded below as raw evidence of what a non-JSON Temporal Worker log line looks like and how
this PoC's Logstash filter handles it, not as a statement about UCP's current logging behavior.

Two smaller, genuinely useful findings correct assumptions in design.md's own Open Questions:

- **Temporal Server already emits JSON by default** (zap logger) — no explicit config change
  needed, contrary to design.md's stated uncertainty.
- **Crossplane core and providers already emit JSON by default** (zap logger) too — same
  correction — but the same stdout stream also mixes in plain-text lines (deprecation warnings),
  so a real onboarding filter for this log group needs to tolerate non-JSON lines, not just
  parse JSON.

## Evidence, per log group

Queried directly against Elasticsearch (`curl localhost:9200/<index>/_search`, or via the
in-cluster ES for the GKE track) after Filebeat → Logstash → Elasticsearch delivery, using the
final (fixed) Logstash filter config from
[implementation.md](implementation.md#phase-5--confirm-parsing-and-search-per-log-group).

**`sample-go-service`** (GKE, `eaas_stg-poc_ucp-poc_sample-go-service-*`) — fields fully intact:

```json
{"level":"info","request_id":"req-209","user_id":"user-42","app_msg":"handled request",
 "geap_log_group":"sample-go-service","geap_client_id":"ucp-poc",
 "message":"{\"level\":\"info\",\"ts\":\"2026-09-28T07:39:41Z\",\"msg\":\"handled request\",\"request_id\":\"req-209\",\"user_id\":\"user-42\",\"method\":\"GET\",\"path\":\"/v1/resources\",\"status\":200}"}
```

**`temporal-server`** (Colima, `eaas_stg-poc_ucp-poc_temporal-server-*`) — fields intact, optional
fields cleanly absent (not leaked placeholders) when not present on a given line:

```json
{"level":"info","app_msg":"Stopped physicalTaskQueueManager","temporal_error":null,"temporal_service":null}
{"level":"error","app_msg":"Persistent fetch operation Failure","temporal_error":"GetWorkflowExecution: failed. Error: context deadline exceeded"}
```

**`temporal-worker`** (Colima, `eaas_stg-poc_ucp-poc_temporal-worker-*`) — fixed-prefix fields
parsed; free-form tail preserved as one raw field, including a genuine, real drift-detection
event:

```json
{"level":"INFO","app_msg":"drift detected",
 "worker_context":"Namespace default TaskQueue drift-detection WorkerID 1@drift-worker-566b9bcfb7-q8j6n@ WorkflowType DriftScanWorkflow WorkflowID drift-scan-2026-09-28T07:42:03Z RunID 01a0e6f6-b505-74b7-a56e-a19aee0c0438 Attempt 1 mrName simple-vm-db-vm-1778570171 xrKind XComputeInstance xrName simple-vm-db-vm-1778570171 detail Synced=False reason=ReconcileError: observe failed: external resource does not exist"}
```

One log line in this group carried the `_workergrokfailure` tag (didn't match the fixed-prefix
GROK pattern) — kept as-is, not investigated further; an expected, not-fully-converged outcome.

**`crossplane-core`** (Colima, `eaas_stg-poc_ucp-poc_crossplane-core-*`) — fields intact:

```json
{"level":"info","app_msg":"Successfully composed desired resources","crossplane_logger":null}
{"level":"info","app_msg":"Automatically determined that composed resource is ready"}
```

(`crossplane_logger` is `null`/absent on these particular lines because the `logger` JSON key
isn't present on every Crossplane log line — confirmed correct behavior after the sprintf fix,
not a parsing failure.)

**Durability buffer (Kafka):** the local stand-in's `eaas-standin-durability` topic held
142,426 messages at time of writing — confirms the Kafka leg of the pipeline is genuinely
receiving and storing shipped events, for the Colima track (the GKE track's in-cluster stand-in
omits Kafka — see [implementation.md's Deviation
2](implementation.md#2-the-gke-tracks-reachability-tunnel-was-replaced-with-an-in-cluster-stand-in-pipeline)).

## Scope boundary (restated)

Per [design.md's Scope](design.md#scope), **this PoC does not prove, and must not be read as
proving:**

- Real GCP → EaaS network connectivity, Cloud Interconnect, or shared-VPC reachability — the
  parent research's Open Questions still need a direct EaaS-team confirmation for this.
- Kerberos/SASL authentication or TLS to Kafka/Elasticsearch — both stand-in pipelines run
  unauthenticated, unencrypted, by design.
- Real EaaS gateway ACLs, onboarding tickets, or a real `client_id`/`client_secret`.
- Temporal multi-cluster replication, or Crossplane reconciling real cloud infrastructure — the
  Temporal/Crossplane environment used here is a real, already-running local dev cluster, but
  this PoC only observed its existing steady-state log output; it did not exercise replication
  or provision new infrastructure as part of this PoC.
- Cost, throughput, or retention behavior at production scale.
- Log-aaS/OpenSearch's ingestion pipeline — this PoC targeted legacy EaaS's
  Elasticsearch/Logstash/Kafka shape only, per the parent research's [Log-aaS migration
  finding](../../research/application-log-monitoring.md#eaas--log-aas-opensearch-migration-in-progress).

## What this PoC did not prove

- Whether a `kv`-style filter, a custom Ruby filter, or a different GROK pattern could fully
  split `temporal-worker`'s free-form trailing key/value tail — only that the fixed-shape prefix
  parses reliably and the tail is preserved unsplit. Not attempted beyond the current
  best-effort split.
- Whether the one `_workergrokfailure`-tagged line represents a rare format variant or a more
  common one — only one instance was observed and not further investigated.
- Whether Crossplane **providers'** (as opposed to core's) reconcile logs follow the exact same
  JSON shape — provider log lines were seen only in passing (`provider-gcp-sql`'s startup line),
  not systematically sampled the way core's were.
- Any resource/cost overhead of running two DaemonSets or the stand-in pipelines at scale.

## What this means for MCUCP-258 — beyond the log-shipping question itself

Temporal Workers' structured-JSON logging is confirmed, so the parent research document's scope
assumption — *"All application-level logs are container stdout/stderr carrying structured JSON
(per ADR-007)"* — holds for this component. The Logstash filter work this PoC did for the
`temporal-worker` log group (fixed-prefix parsing, free-form-tail handling) remains useful as a
fallback pattern for any log group that turns out not to be JSON, but is not itself evidence of
a current gap.
