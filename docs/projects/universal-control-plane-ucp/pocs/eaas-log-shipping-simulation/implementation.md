---
title: "EaaS Log Shipping Simulation — Implementation"
space: UCP
parent_page_id: "../eaas-log-shipping-simulation.md"
---

# EaaS Log Shipping Simulation — Implementation

Supporting proof doc. Describes what was built, run, and observed for each phase in
[design.md](design.md). See [poc-report.md](poc-report.md) for the verdict.

## Deviations from design.md, and why

Two adaptations were made during execution. Both are scope-preserving — neither changes what
this PoC proves or does not prove (see [design.md's Scope](design.md#scope)) — but both change
*how* the design's phases were carried out.

### 1. Temporal Server, Temporal Worker, and Crossplane were captured from a live, pre-existing
local cluster, not the planned MCR-PoC/crossplane-xrd-catalog-source-PoC reuse

design.md's Phase 3b planned to stand up a fresh single-cluster Temporal Server (reusing the
[Temporal Multi-Cluster Replication PoC](../temporal-multi-cluster-replication/implementation.md)'s
docker-compose config) and reuse the
[Crossplane XRD Catalog Source PoC](../crossplane-xrd-catalog-source/implementation.md)'s
`ucp-crossplane` Colima/k3s setup with a dummy/no-op XRD.

On starting Colima's **`default`** profile (needed as a Docker host for the stand-in pipeline),
its k3s cluster turned out to already be running a long-lived (175-day-old), much more
representative local environment: a real Helm-deployed Temporal Server (chart
`temporal-0.74.0`, components `frontend`/`history`/`matching`/`worker`/`web`/`admintools`),
UCP's own real Temporal Workers (`ucp-provisioning-worker`, `drift-worker`), and Crossplane 2.2.0
with real GCP providers (`provider-gcp-sql`, `-dns`, `-compute`, `-storage`,
`-servicenetworking`, `-container`, `upbound-provider-family-gcp`) actively reconciling real
composite resources (an `XComputeInstance` drift scan produced a genuine "external resource does
not exist" log line during this run).

**This is used instead of the planned synthetic substitutes** — it is materially more
representative of UCP's actual log shapes (real component names matching the parent research's
[Scope: components to
log](../../research/application-log-monitoring.md#scope-components-to-log) table exactly), and
required zero new setup beyond deploying the Filebeat DaemonSet. The MCR PoC's and
crossplane-xrd-catalog-source PoC's own environments were left untouched.

**Consequence for the shipper count:** because Temporal Server, Temporal Worker, and Crossplane
core all turned out to already run in the *same* k3s cluster (rather than Temporal being a
separate docker-compose stack per design.md's Phase 3c), **one Filebeat DaemonSet covers all
three log groups** via metadata-based routing, not the two separate Colima-side shippers
design.md planned (a k3s DaemonSet for Crossplane + a shared Filebeat for the Temporal
docker-compose stack). Total shipper count across both tracks: **2** (one GKE DaemonSet, one
Colima-k3s DaemonSet), covering all 4 log groups — an even flatter shape than design.md's planned
3, while still fully honoring "shared shipper, not sidecar-per-pod."

### 2. The GKE track's reachability tunnel was replaced with an in-cluster stand-in pipeline

design.md's Phase 2 planned a `cloudflared`/`ngrok` TCP tunnel from the sandbox GKE cluster to
the single local docker-compose stand-in pipeline. Neither worked in this environment:

- **ngrok**: refused to start a TCP tunnel — `ERR_NGROK_4018`, "ngrok requires an account and a
  valid credential to start a session." No ngrok account/authtoken is available in this
  environment.
- **cloudflared**: a `cloudflared tunnel --url tcp://localhost:5044` "quick tunnel" was created
  successfully, but the client-side proxy needed to actually use it
  (`cloudflared access tcp --hostname <tunnel> --url ...`) failed with `websocket: bad handshake`
  against the tunnel origin — reproduced twice, with two independently fresh quick tunnels, both
  from inside a GKE pod and from the local machine directly. This appears to be a limitation of
  pairing anonymous TryCloudflare quick tunnels with `access tcp` (which likely expects a named
  tunnel + Cloudflare Access application, requiring a Cloudflare account) rather than a config
  mistake — not independently confirmed against Cloudflare's own documentation/support, since
  that would require the same missing account.

**Adaptation:** the GKE track's stand-in pipeline was deployed **in-cluster**, inside the sandbox
GKE cluster's `eaas-poc` namespace (Elasticsearch + Logstash + Kibana — Kafka omitted for this
instance only, see below), removing the need for any tunnel. This still fully satisfies
design.md's actual goal for the GKE track — proving a Filebeat DaemonSet running in the sandbox
GKE cluster can ship the `sample-go-service` log shape through a Logstash/Elasticsearch/Kibana
pipeline shaped like EaaS and get it parsed correctly — it only changes *where* that pipeline
physically runs, not what is being tested. Kafka's durability-buffer role is still fully
exercised, just by the Colima track's connection to the original local docker-compose pipeline
(which does include Kafka) rather than by this in-cluster instance.

---

## Source Code

- **Repository:** `aripermana-putra/kitchen-sink`
- **Path:** `eaas-log-shipping-simulation-poc/`
- **Files:**
  - `eaas-standin/docker-compose.yml`, `eaas-standin/logstash/logstash.yml`,
    `eaas-standin/logstash/pipeline/eaas-standin.conf` — the local docker-compose stand-in
    pipeline (Logstash + Kafka + Elasticsearch + Kibana), reached directly by the Colima track
  - `gke/00-namespace.yaml`, `01-sample-app.yaml`, `03-filebeat-daemonset.yaml`,
    `04-eaas-standin-configmap.yaml`, `05-eaas-standin-workloads.yaml` — the GKE track: sample
    workload, Filebeat DaemonSet, and the in-cluster stand-in pipeline (Elasticsearch + Logstash
    + Kibana, no Kafka)
  - `colima-k8s/filebeat-daemonset.yaml` — the shared Filebeat DaemonSet for the Colima
    `default`-profile k3s cluster (Temporal Server, Temporal Worker, Crossplane core)
- **Not committed** — left as uncommitted working-tree files, per the user's global commit
  discipline. See [Commit note](#commit-note-what-would-need-to-land) at the end of this doc.

---

## Environment

| | |
|---|---|
| **Colima profile hosting the local stand-in pipeline** | `ucp-crossplane` (4GB/2CPU) — chosen because it was idle; the profile's own Crossplane/k3s installation was *not* used for this PoC (see Deviation 1) |
| **Colima profile hosting Temporal Server / Worker / Crossplane** | `default` (8GB/4CPU) — the pre-existing, already-running k3s cluster (kubectl context `colima`) |
| **GKE cluster** | `ucp-agent-cluster`, project `sub-gcp-ucp-clsd-sandbox`, namespace `eaas-poc` (created for this PoC) |
| **Local device** | Runs the `eaas-standin` docker-compose stack; must stay running for the Colima track to reach it |

---

## Phase-by-phase record

### Phase 1 — Local stand-in pipeline

`docker compose up -d` in `eaas-standin/` on the `ucp-crossplane` Colima profile. Image
substitution was required: `bitnami/kafka:3.7` no longer resolves (`docker.io/bitnami/kafka:3.7:
not found` — Bitnami retired that tag); replaced with `apache/kafka:3.7.0` in KRaft mode
(controller+broker combined, no Zookeeper), which came up cleanly. Elasticsearch/Logstash/Kibana
7.17.18, heap capped at 512m each (`ES_JAVA_OPTS`/`LS_JAVA_OPTS`) to fit the 4GB profile. All four
containers healthy on first successful compose run (`_cluster/health` → `green`; Logstash's
`beats` input listening on `:5044`).

### Phase 2 — Reachability tunnel (GKE track)

See [Deviation 2](#2-the-gke-tracks-reachability-tunnel-was-replaced-with-an-in-cluster-stand-in-pipeline)
above — both `ngrok` and `cloudflared` failed for reasons unrelated to shipper/parsing
correctness. No tunnel exists in the final setup; the GKE track uses an in-cluster stand-in
pipeline instead (Phase 3's `04`/`05` manifests).

### Phase 3 — Sample workload + Filebeat DaemonSet (GKE)

`sample-go-service` deployed as a `busybox` shell loop (not a compiled Go binary — this PoC
tests log *shape* ingestion, not app logic) emitting ADR-007 `slog`-shaped JSON every 3s, with a
synthetic `error`-level line every 5th iteration.

Filebeat deployed as a genuine DaemonSet (`ServiceAccount` + `ClusterRole`/`ClusterRoleBinding`
for `namespaces`/`pods`/`nodes` `get`/`list`/`watch`, needed by `add_kubernetes_metadata`) —
3 pods, one per `system-pool` node, confirmed `Running` with no RBAC errors.

**Two real bugs found and fixed during this phase and Phase 5, both directly relevant to a real
EaaS onboarding, not just this PoC:**

1. **Unscoped `container` input harvests every pod's logs on the node.** The first DaemonSet
   revision used `paths: ["/var/log/containers/*.log"]` with a `log_group` condition keyed only
   on `kubernetes.namespace`. Because Filebeat's `container` input reads *every* container log
   file on the node by default, and this GKE node had been running many unrelated pods for days,
   the DaemonSet immediately harvested and shipped their historical backlog too. Since those
   events have no matching `log_group`, the ES output's `%{[geap_log_group]}` index-name
   interpolation left the **literal, unresolved sprintf token** in the index name
   (`eaas_stg-poc_ucp-poc_%{[fields][geap][log_group]}-*`) instead of failing loudly — thousands
   of docs landed in a garbage index before this was caught. **Fixed** by (a) scoping
   `paths` to the target namespace/pod glob and (b) adding an explicit `else: drop_event: ~`
   to the routing conditional, so anything not matching a known log_group is dropped outright
   rather than silently mis-indexed. This is exactly the "misconfigured routing rule could
   silently misroute logs" risk design.md's Risks table anticipated — confirmed here as a real
   failure mode, not a hypothetical one.
2. **Filebeat's `add_fields` processor with a dotted string `target` does not nest into an
   existing structure.** The first fix attempt conditionally added `fields.geap.log_group` via
   `add_fields: {target: "fields.geap", fields: {log_group: ...}}`, alongside a *separate*
   top-level static `fields: {geap: {client_id, client_secret}}` block. Inspecting the raw
   indexed document showed **two sibling keys** — a properly nested `fields.geap` (from the
   static block, holding only `client_id`/`client_secret`) and a second, literally-named
   top-level key `"fields.geap"` (from the processor, holding only `log_group`) — not merged.
   Logstash's `[fields][geap][log_group]` reference therefore never resolved, and its sprintf
   output again left the literal unresolved token in the field value. **Fixed** by moving
   `client_id`/`client_secret`/`log_group` into a *single* `add_fields` call with `target: fields`
   and a properly nested `fields: {geap: {...}}` mapping, all set together inside the same
   conditional. This is a genuinely subtle Filebeat gotcha — worth flagging for real EaaS
   onboarding, since it would silently corrupt log-group routing there too, with no error at
   either the Filebeat or Logstash layer.

### Phase 3b — Temporal Server, Temporal Worker, Crossplane core (Colima)

No fresh deployment — see Deviation 1. Real log samples pulled directly via `kubectl logs` to
inform the Logstash filter design *before* wiring up Filebeat:

- **Temporal Server already emits JSON by default** (zap logger): `{"level":"error","ts":"...",
  "msg":"...","error":"...", "logging-call-at":"...", "stacktrace":"..."}` — **no config change
  needed.** This corrects design.md's Open Questions assumption that JSON encoding might need to
  be explicitly configured.
- **Crossplane core and providers also emit JSON by default** (zap): `{"level":"info","ts":"...",
  "logger":"crossplane","msg":"..."}` — again, **no config change needed.** But the same stdout
  stream also carries plain-text lines with no JSON structure at all (e.g. `Warning:
  apiextensions.crossplane.io Usage is deprecated; migrate to ...`) — a real mixed-format
  stream, not pure JSON.
- **Temporal Worker (UCP's own `ucp-provisioning-worker`/`drift-worker`) is plain text, not
  JSON**, via the Temporal Go SDK's default logger: `2026/09/28 07:22:04 INFO  Drift worker
  starting. addr=... tq=drift-detection` and, for SDK-internal activity/workflow diagnostics,
  lines like `2026/09/28 07:23:29 DEBUG ExecuteActivity Namespace default TaskQueue
  drift-detection WorkerID 1@drift-worker-...@ WorkflowType DriftScanWorkflow WorkflowID
  drift-scan-... RunID ... Attempt 1 ActivityID 17 ActivityType ScanDriftActivity` — a
  fixed-shape `date time LEVEL  message` prefix followed by a **variable number of
  space-separated `Key Value` token pairs with no `=` delimiter**. This is a significant, real
  finding — see [poc-report.md](poc-report.md) for why it matters beyond this PoC.

Filebeat deployed as one DaemonSet (`eaas-poc` namespace, created fresh in this cluster too),
`paths` scoped to `/var/log/containers/*_temporal-system_*.log` and
`*_crossplane-system_*.log`, with a three-way `if`/`else-if`/`else-if`/`else: drop_event`
routing chain keyed on `kubernetes.labels.app_kubernetes_io/name: "temporal"` (→
`temporal-server`), `kubernetes.labels.app` being `ucp-provisioning-worker` or `drift-worker`
(→ `temporal-worker`), and `kubernetes.namespace: "crossplane-system"` (→ `crossplane-core`).
Reused the same `add_fields`/`target: fields` nesting fix from Phase 3. 1 pod (this cluster has
1 node), `Running`, no RBAC errors.

### Phase 3c — Reachability (Colima)

Confirmed directly with a throwaway `busybox` pod in the `colima` cluster: `nc -z -w3 192.168.5.2
5044` → `REACHABLE` — Colima's `vz` driver guest gateway routes to the Mac host, which forwards
the `ucp-crossplane` profile's Logstash port to `localhost:5044`, exactly the cross-VM networking
pattern already documented in the
[Temporal Multi-Cluster Replication PoC](../temporal-multi-cluster-replication/implementation.md#phase-12--stand-up-both-clusters).
No tunnel needed for this track, as design.md anticipated.

### Phase 4 — Confirm shipping

Both shippers reached `Connection ... established` against their respective Logstash targets
with no persistent errors in steady state (`filebeat` monitoring metrics showed `failed: 0` once
connected). Filebeat's monitoring logs on the Colima DaemonSet did show transient
`client is not connected` / reconnect cycles immediately after each Logstash restart (expected —
Logstash's beats server issues a new session on restart) but these self-resolved within ~1s.

### Phase 5 — Confirm parsing and search, per log group

All four log groups confirmed present, correctly field-parsed, and distinguishable by
`geap_log_group` — see [poc-report.md](poc-report.md#evidence-per-log-group) for the actual
query results. Two Logstash-side parsing gaps were found and fixed here (see below); one
`temporal-worker` parsing gap was found and **not** fixed (deliberately left as a recorded
finding, per design.md's risk-handling guidance):

- **Sprintf leaves unresolved tokens for absent optional fields.** `temporal_service` (only
  present on some Temporal Server log lines) and `crossplane_logger` (only present on some
  Crossplane log lines) were being set via unconditional
  `mutate { add_field => { "x" => "%{[parsed][y]}" } }`. When `[parsed][y]` didn't exist on a
  given event, Logstash's sprintf left the literal string `%{[parsed][y]}` in the field —not an
  empty string, not a missing field. **Fixed** by wrapping each optional field in its own
  `if [parsed][y] { ... }` existence guard. Confirmed fixed: re-queried both log groups
  afterward and the fields are now cleanly absent (`null`) rather than carrying the literal
  placeholder.
- **`temporal-worker`'s free-form trailing key/value tokens are not fully split, by design.**
  The GROK filter extracts the fixed `date time LEVEL  message` prefix reliably. The variable
  trailing `Key Value Key Value ...` tail (no `=` delimiter, so Logstash's `kv` filter doesn't
  apply) is kept as one raw `worker_context` string rather than split into individual fields —
  this was the intended, documented scope from design.md, not a bug to fix. One log line was
  also observed with a `_workergrokfailure` tag (didn't match the fixed-prefix pattern at all) —
  left as-is, an expected "not everything converges on the first pass" outcome design.md's
  Hypothesis anticipated.

### Phase 6 — Record findings

Captured throughout this document and [poc-report.md](poc-report.md); Kerberos/SASL, TLS, real
EaaS gateway ACLs, GCP Cloud Interconnect, Temporal multi-cluster replication, and real
Crossplane provider infrastructure provisioning were not exercised, per design.md's scope — see
[poc-report.md's Scope boundary](poc-report.md#scope-boundary-restated) for the explicit
restatement.

---

## Current running state (as of this PoC run)

Left running, not torn down, in case further inspection is wanted:

| Component | Where | State |
|---|---|---|
| `eaas-standin` docker-compose stack | `ucp-crossplane` Colima profile | Running (`localhost:9200` ES, `:5601` Kibana, `:5044` Logstash beats input) |
| Filebeat DaemonSet + sample workload + in-cluster stand-in pipeline | GKE `eaas-poc` namespace | Running |
| Filebeat DaemonSet | Colima `default` profile, `eaas-poc` namespace | Running |
| `colima` (`default`) and `ucp-crossplane` profiles | — | Both `Running` (were `Stopped` before this PoC) |

## Commit note: what would need to land

Nothing has been committed. If this work is kept:

- `eaas-log-shipping-simulation-poc/` in `kitchen-sink` — new directory, all files untracked.
- This doc, [poc-report.md](poc-report.md), [evidence.md](evidence.md), and
  [../eaas-log-shipping-simulation.md](../eaas-log-shipping-simulation.md)'s status field update,
  in the docs repo.
