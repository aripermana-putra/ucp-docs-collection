---
title: "EaaS Log Shipping Simulation — Design"
space: UCP
parent_page_id: "../eaas-log-shipping-simulation.md"
---

# EaaS Log Shipping Simulation — Design

Human review document. Read this before the PoC starts executing.

| | |
|---|---|
| **Ticket** | MCUCP-258 |
| **Parent research** | [Application Log Monitoring — Cloud Logging (GCP) vs EaaS vs Self-Hosted EFK](../../research/application-log-monitoring.md) |
| **Related PoC** | No Option A/C counterpart — Cloud Logging (Option A) and self-hosted EFK (Option C) are not being PoC'd per the parent research's [Related PoCs](../../research/application-log-monitoring.md#related-pocs) section. This PoC reuses infrastructure patterns (not findings) from the [Temporal Multi-Cluster Replication PoC](../temporal-multi-cluster-replication.md) and the [Crossplane XRD Catalog Source PoC](../crossplane-xrd-catalog-source.md) to get real Temporal/Crossplane log sources without deploying them from scratch |

---

## Research Question

> Can a Filebeat or Vector shipper, configured per EaaS's documented conventions (client_id/
> client_secret, `geap_log_group` tagging, BEAT-protocol output), correctly ship UCP's actual
> log shapes — not just a generic Go service's structured JSON, but also Temporal Server's,
> Temporal Worker's, and Crossplane's own log output — through a pipeline that mirrors EaaS's
> own architecture (Logstash → Kafka → Elasticsearch → Kibana), when that pipeline is simulated
> locally instead of being the real EaaS service?

This is a narrower question than "does Option B work for UCP." The parent research's [Option B
findings](../../research/application-log-monitoring.md#option-b--eaas-rakuten-onecloud-logging-platform)
already show, via other Rakuten GCP-hosted services' Confluence-documented history, that the
*network* side of Option B is confirmed working in principle. What has not been exercised at all
is whether UCP-shaped shipper configs — pointed at a pipeline shaped like EaaS's own — actually
produce correctly parsed, searchable log entries for **every distinct log shape UCP's own
component footprint produces**, not just a single representative Go service. That is what this
PoC measures. Real GCP→EaaS connectivity is explicitly out of scope (see [Scope](#scope) and
[Open Questions](#open-questions)) and is tracked as a direct confirmation with the EaaS team in
the parent research document, not something this PoC attempts.

The parent research's [Scope: components to log](../../research/application-log-monitoring.md#scope-components-to-log)
table lists several component *types*, not one uniform shape:

- UCP's own Go services (API Server, Temporal Workers) — these emit ADR-007's `slog`-based
  structured JSON, since UCP controls their logging code directly.
- **Temporal Server** (Frontend, History, Matching, Internal Worker) — a third-party binary
  (`temporalio/auto-setup` in other UCP PoCs) using its own zap-based server logging config, not
  UCP's `slog` code.
- **Temporal Workers** (Provisioning, Drift) — UCP-authored Go binaries, but built on the
  Temporal Go SDK, which has its own pluggable `log.Logger` interface; the SDK's own internal
  diagnostic messages (retries, task timeouts) flow through whatever logger UCP wires in, and may
  not automatically match ADR-007's exact field shape even if UCP's own log lines do.
- **Crossplane core + providers** (`roc-lbaas/vmaas/dbaas/staas/caas`, `upjet-gcp`,
  `gcp-sql/container/compute/storage`) — third-party `controller-runtime`-based Go binaries,
  using their own zap-based reconcile logging, not ADR-007's convention.

If EaaS's onboarding (or any option's ingestion mechanism) needs a different GROK/JSON parsing
pattern per log shape — likely, since EaaS's own documentation says parsing patterns are
hand-tuned per log group — then "does the pipeline parse UCP's logs correctly" is really three
or four separate questions, not one. This PoC answers all of them, not just the Go-service one.

---

## Hypothesis

**Filebeat is deployed as a shared, per-environment shipper (a DaemonSet on each Kubernetes
cluster; a single shared container tailing Docker's log driver on the Colima docker-compose
Temporal stack), not as a sidecar container per pod/component.** This mirrors how GKE's own
Cloud Logging agent, the self-hosted EFK option's Fluent Bit DaemonSet, and CaaS's own shared
shipping path all work — one shipper per node/environment, disambiguating log source via
metadata rather than via a dedicated container per workload. See [Scope](#scope) for exactly
which shipper covers which components.

This shared shipper, configured with the same `fields.geap` block, `geap_log_group` tagging
(assigned dynamically per entry from pod/container metadata, not a static per-shipper value),
and `output.logstash` BEAT-protocol convention documented in EaaS's own onboarding guides (see
[evidence.md](evidence.md)), can ship each of UCP's real log shapes — a generic Go service's
ADR-007 `slog` JSON, Temporal Server's own server logs, a Temporal Worker's SDK-wrapped logs, and
a Crossplane controller's reconcile logs — to a locally-run, docker-composed stand-in for EaaS's
pipeline (Logstash → Kafka → Elasticsearch → Kibana), and produce log entries in Kibana with:

- each log shape's meaningful fields intact and individually queryable — not flattened into a
  single opaque `message` string, even though the four shapes' field names and structures differ
  from each other, and
- the shipper-added metadata (`geap_log_group`, `client_id`, tags) present and correctly
  attributed, distinguishing which component/log-group each entry came from,

without needing any real EaaS credentials, gateway hostnames, or GCP→DC-network connectivity.
Getting every shape right on the first pass is not expected — EaaS's own documentation notes
message-parsing patterns are typically hand-tuned per log group, and Temporal Server/Crossplane's
default log encoding is not guaranteed to be JSON out of the box — but the PoC should converge on
a working config per shape, and any parsing gaps found here are a direct preview of what UCP
would hit during real EaaS onboarding for that specific component, independent of whether the
network path exists.

---

## Scope

| Item | In scope | Out of scope |
|------|----------|--------------|
| Sandbox GKE cluster (`ucp-agent-cluster`, `sub-gcp-ucp-clsd-sandbox`) hosting a sample Go-service log-generating workload | ✅ | |
| Sample workload emitting structured JSON stdout/stderr matching ADR-007's `slog` convention (`level`, `request_id`, `user_id`, timestamp, message) | ✅ | |
| **Temporal Server**, deployed locally via Colima/docker-compose reusing the pattern already proven in the [Temporal Multi-Cluster Replication PoC](../temporal-multi-cluster-replication.md) (`temporalio/auto-setup` image, Postgres persistence, `ucp-crossplane` or a dedicated Colima profile) — real server logs captured, not synthetic | ✅ | |
| **A Temporal Worker** (Go SDK), reusing/adapting the [Temporal Multi-Cluster Replication PoC](../temporal-multi-cluster-replication.md)'s `worker/main.go` — real SDK-wrapped worker logs (workflow/activity execution, retries) captured, not synthetic | ✅ | |
| **Crossplane core**, deployed locally via Colima/k3s reusing the pattern already proven in the [Crossplane XRD Catalog Source PoC](../crossplane-xrd-catalog-source.md) (`ucp-crossplane` Colima profile) — real controller-runtime reconcile logs captured, not synthetic; at least one dummy/no-op XRD reconciling is sufficient, a real cloud provider is not required | ✅ | |
| **One Filebeat DaemonSet on the sandbox GKE cluster** (per-node, not per-pod), using `add_kubernetes_metadata` and a conditional processor to set `fields.geap.log_group` per entry from pod/namespace/label metadata | ✅ | |
| **One Filebeat DaemonSet on the Colima k3s cluster** hosting Crossplane core (per-node, not per-pod) — same metadata-based `log_group` assignment; ready to cover additional Crossplane provider pods later without adding shippers | ✅ | |
| **One shared Filebeat container/process on the Colima docker-compose Temporal stack**, tailing both Temporal Server's and Temporal Worker's container logs via Docker's log driver (not one Filebeat per container), setting `log_group` from container name | ✅ | |
| Every shipper configured with `fields.geap` (`client_id`, `client_secret`), tags, and `output.logstash` pointed at the simulated pipeline — `log_group` set dynamically per entry as above, not hardcoded per shipper | ✅ | |
| A Filebeat sidecar-per-pod/per-container model | | ✅ — deliberately not used; see [Hypothesis](#hypothesis). Sidecar-per-pod would mean one Filebeat instance per pod replica across UCP's full component footprint (API Server, every Temporal service, every Crossplane provider), which is real resource duplication with no benefit over metadata-based routing on a shared shipper |
| A locally-run (docker-compose) stand-in pipeline: Logstash (`beats` input) → Kafka (durability buffer) → Elasticsearch (single-node, security disabled) → Kibana | ✅ | |
| A reachable endpoint from the sandbox GKE pod to the local stand-in pipeline (tunnel — e.g. `cloudflared` or `ngrok` TCP tunnel exposing Logstash's beats-input port) — only needed for the GKE-hosted sample workload; the Colima-hosted components reach the local pipeline directly over the local network, no tunnel required | ✅ | |
| Per-component Logstash pipeline config (GROK/JSON filters) approximating EaaS's documented hand-tuned-per-log-group pattern, one filter per log shape: Go `slog` JSON, Temporal Server logs, Temporal Worker/SDK logs, Crossplane reconcile logs | ✅ | |
| Confirming shipped logs for all four log groups are visible, correctly field-parsed, and searchable/distinguishable (by `geap_log_group`) in the local Kibana instance, despite being shipped by only three shared shippers | ✅ | |
| Recording shipper-config and parsing gotchas per component (parsing failures, multiline/stack-trace handling, field mapping issues, version-compatibility notes) as they would apply to real EaaS onboarding | ✅ | |
| Configuring Temporal Server's and Crossplane's logging output to JSON (if not the default) — confirming the actual config flag/field during implementation | ✅ | |
| A second variant using Vector instead of Filebeat, shipping directly to the stand-in Kafka (mirroring the GCP-C/Session-Hint-Cookie pattern from the parent research) | | ✅ — stretch goal only if time allows; not required for this PoC's success criteria (see [Open Questions](#open-questions)) |
| A real cloud provider (e.g. `gcp-sql`, `upjet-gcp`) reconciling real infrastructure under Crossplane | | ✅ — a dummy/no-op XRD's reconcile-loop logs are enough to capture Crossplane's real log shape; provisioning real infra is unrelated to this PoC's question |
| Real Temporal multi-cluster replication behavior | | ✅ — this PoC only needs one Temporal Server + one worker running and producing logs, not the replication scenario the source PoC was built for |
| Real EaaS gateway, Kafka brokers, or Elasticsearch/Kibana | | ✅ — this PoC never talks to the real EaaS service |
| A real EaaS `client_id`/`client_secret` or onboarding ticket | | ✅ — the stand-in pipeline requires no real credentials |
| GCP Cloud Interconnect, shared-VPC provisioning, or any real GCP→DC-network path | | ✅ — tracked separately as a direct EaaS-team confirmation in the parent research's Open Questions, not testable until UCP's GCP environment exists |
| Kerberos KDC / SASL_GSSAPI authentication to Kafka (required for the real EaaS Secured tier's direct-Kafka path) | | ✅ — the stand-in Kafka runs unauthenticated; this is a known, deliberate simulation gap, not a finding about the real Secured tier |
| TLS/Secured-tier-equivalent transport security on the stand-in pipeline | | ✅ — stand-in Elasticsearch/Logstash run without TLS; not a claim about EaaS Secured tier's real posture |
| Retention/ILM policy behavior, Kibana SSO/access control, or the "One Kibana" cross-cluster-search feature | | ✅ — none of these are exercised by a single local Elasticsearch node |
| Cost or throughput measurement at production scale | | ✅ — sandbox traffic is not representative |
| Log-aaS / OpenSearch's ingestion pipeline | | ✅ — this PoC targets legacy EaaS's Elasticsearch/Logstash/Kafka shape, since that is what UCP would onboard onto first per the parent research's [Log-aaS migration
finding](../../research/application-log-monitoring.md#eaas--log-aas-opensearch-migration-in-progress) |

---

## Approach

```mermaid
flowchart TD
    subgraph SandboxGKE["Sandbox GKE — ucp-agent-cluster (sub-gcp-ucp-clsd-sandbox)"]
        App[Sample Go workload pod\nstructured JSON stdout/stderr\nADR-007 slog shape]
        FilebeatGKE[["Filebeat DaemonSet\n(one per node, not per pod)\nadd_kubernetes_metadata +\nconditional log_group processor"]]
        App -- "stdout/stderr,\nread from node's\nkubelet log dir" --> FilebeatGKE
    end

    Tunnel[Reachability tunnel\ncloudflared / ngrok TCP tunnel\nexposes local Logstash beats-input port]

    subgraph ColimaEnv["Local Colima environment — reusing existing PoC patterns"]
        direction TB
        subgraph TemporalStack["ucp-crossplane or temporal-mcr profile, docker-compose\n(Temporal Multi-Cluster Replication PoC pattern)"]
            TServer[Temporal Server container\ntemporalio/auto-setup\nzap server logs]
            TWorker[Temporal Worker container\nGo SDK, worker/main.go\nSDK-wrapped logs]
            FilebeatTemporal[["Filebeat — one shared container\ntailing Docker's log driver for\nboth Temporal containers\nlog_group set from container name"]]
            TServer -- "stdout/stderr" --> FilebeatTemporal
            TWorker -- "stdout/stderr" --> FilebeatTemporal
        end
        subgraph CrossplaneStack["ucp-crossplane profile, k3s\n(Crossplane XRD Catalog Source PoC pattern)"]
            Crossplane[Crossplane core pod(s)\ncontroller-runtime\nreconcile logs\ndummy/no-op XRD]
            FilebeatCrossplane[["Filebeat DaemonSet\n(one per k3s node, not per pod)\nsame metadata-based routing\nas the GKE DaemonSet"]]
            Crossplane -- "stdout/stderr,\nread from node's\nkubelet log dir" --> FilebeatCrossplane
        end
    end

    subgraph LocalDevice["Local device — simulated EaaS stand-in (docker-compose)"]
        Logstash[Logstash\nbeats input :5044\nper-log_group GROK/JSON filters]
        Kafka[(Kafka\ndurability buffer)]
        ES[(Elasticsearch\nsingle-node, no TLS/auth)]
        Kibana[Kibana\nDiscover UI]
        Logstash --> Kafka --> ES --> Kibana
    end

    FilebeatGKE -- "output.logstash,\nBEAT protocol" --> Tunnel --> Logstash
    FilebeatTemporal -- "output.logstash,\nlocal network, no tunnel" --> Logstash
    FilebeatCrossplane -- "output.logstash,\nlocal network, no tunnel" --> Logstash

    P1[Phase 1\nStand up local\nsimulated EaaS pipeline] --> P2[Phase 2\nExpose pipeline via tunnel\nfor the GKE track]
    P2 --> P3[Phase 3\nDeploy sample workload +\nFilebeat DaemonSet in sandbox GKE]
    P1 --> P3b[Phase 3b\nStand up Temporal Server +\nWorker + Crossplane in Colima]
    P3b --> P3c[Phase 3c\nAttach one shared Filebeat\nper environment in Colima]
    P3 --> P4[Phase 4\nConfirm shipping\nall three shippers]
    P3c --> P4
    P4 --> P5[Phase 5\nConfirm per-log_group\nparsing + search in Kibana]
    P5 --> P6[Phase 6\nRecord per-component\nfindings]
    P5 -->|Fields not parsed\nor no data, per log_group| Investigate[Investigate Logstash filter\nor log_group routing rule]
    Investigate --> P5
```

### Phase 1 — Stand up the local simulated EaaS pipeline

1. Write a `docker-compose.yml` running Logstash (with a `beats` input listening on a port
   matching EaaS's documented convention shape, e.g. `5044`), Kafka (single broker, no SASL —
   durability buffer only, matching EaaS's own architecture diagram), Elasticsearch
   (single-node, `xpack.security.enabled: false`), and Kibana.
2. Confirm all four containers are healthy and Kibana can reach Elasticsearch before moving on.

### Phase 2 — Expose the pipeline to the sandbox GKE pod

1. Start a `cloudflared` or `ngrok` **TCP** tunnel (not HTTP — Filebeat's `output.logstash`
   speaks the raw BEAT protocol over TCP) pointed at the local Logstash beats-input port.
2. Record the tunnel's public hostname:port — this is a PoC-only artifact standing in for a real
   EaaS gateway endpoint and must be clearly labeled as such in `implementation.md`, not
   confused with a real network path. This tunnel is only needed for the GKE-hosted sample
   workload — the Colima-hosted components in Phase 3b/3c reach the same local Logstash port
   directly, since they already run on the same machine (or a VM on it) as the simulated
   pipeline.

### Phase 3 — Deploy the sample Go workload and a Filebeat DaemonSet in the sandbox cluster

1. Deploy a sample workload emitting structured JSON stdout/stderr matching ADR-007's `slog`
   shape (`level`, `request_id`, `user_id`, timestamp, message) — reuse or adapt a workload
   already used in the metrics PoCs if one emits comparable log output, otherwise a small
   purpose-built log generator.
2. Deploy Filebeat as a **DaemonSet** (one pod per node, reading every container's logs from the
   node's kubelet log directory — not a sidecar in the sample workload's own pod), configured
   with:
   - `add_kubernetes_metadata` to enrich each entry with pod/namespace/label metadata.
   - A conditional processor (e.g. `if`/`add_fields` keyed on namespace or a label like
     `ucp.io/log-group`) that sets `fields.geap.log_group` per entry — for this PoC, everything
     in the sample workload's namespace maps to `log_group: sample-go-service`, but the
     mechanism is designed to add more namespaces/labels later without adding shippers.
   - `fields.geap.client_id` / `client_secret` — synthetic values, matching the shape EaaS's
     onboarding guides document, not real credentials.
   - `tags` for environment filtering, per EaaS's documented convention.
   - `output.logstash.hosts` pointed at the tunnel endpoint from Phase 2.
3. Confirm `filebeat test config` passes before starting the DaemonSet, and confirm the DaemonSet
   has the RBAC/hostPath access it needs to read node-level container logs (this may require a
   `ClusterRole` for `add_kubernetes_metadata` and a `hostPath` volume mount — record any
   permission setup as a real finding, the same way the MonaaS PoC recorded RBAC setup for its
   receivers).

### Phase 3b — Stand up Temporal Server, a Temporal Worker, and Crossplane core in Colima

1. **Temporal Server**: reuse the [Temporal Multi-Cluster Replication PoC](../temporal-multi-cluster-replication/implementation.md)'s
   `docker-compose.yml`/`config_template.yaml` (image `temporalio/auto-setup:1.25.2`, Postgres
   13 persistence) inside the existing `ucp-crossplane` Colima VM, or a fresh dedicated profile
   if resource contention with the Crossplane stack becomes an issue — only one Temporal cluster
   is needed here, not the source PoC's two-cluster replication setup.
2. Confirm Temporal Server's logging config: check whether its YAML config exposes a
   `log.encoding` (or equivalent) field to select JSON output, since the source PoC's config
   didn't need to care about this. If JSON isn't the default, set it explicitly and record the
   config change — this is itself a finding, not a workaround.
3. **Temporal Worker**: reuse/adapt the source PoC's `worker/main.go` (Go SDK), pointed at this
   Temporal Server, running one simple workflow+activity on a loop (or on manual trigger) so it
   produces a steady stream of real worker logs (task polling, workflow/activity execution,
   retries) rather than a one-shot run.
4. Confirm what logger the worker's `worker.Options.Logger` is wired to (the Temporal Go SDK's
   default, or a custom adapter) and whether its output is JSON — record this, since it
   determines how much of ADR-007's `slog` convention actually carries through SDK-originated log
   lines versus UCP's own log lines within the same worker process.
5. **Crossplane core**: reuse the [Crossplane XRD Catalog Source PoC](../crossplane-xrd-catalog-source/implementation.md)'s
   `ucp-crossplane` Colima/k3s setup (Crossplane already installed there) and apply one
   dummy/no-op XRD so its controller produces real reconcile-loop log lines on a steady interval
   — no real cloud provider or infrastructure provisioning is needed.
6. Confirm Crossplane/controller-runtime's logging flags: check whether the deployed Crossplane
   version supports a JSON-encoder flag for its zap logger (e.g. via `--zap-encoder` or
   equivalent — confirm the actual flag name against the deployed version rather than assuming
   it) and set it if JSON isn't already the default.

### Phase 3c — Attach one shared Filebeat per environment in Colima (not one per component)

1. **Crossplane's k3s cluster**: deploy a Filebeat **DaemonSet**, the same model as Phase 3's
   GKE DaemonSet — `add_kubernetes_metadata` plus a conditional processor mapping Crossplane's
   namespace/labels to `log_group: crossplane-core`. This is a genuine DaemonSet since Crossplane
   runs under k3s (real Kubernetes), unlike the Temporal stack below.
2. **Temporal's docker-compose stack**: docker-compose has no DaemonSet concept, but the same
   "one shared shipper, not one per container" principle applies — run a **single** Filebeat
   container tailing both the Temporal Server and Temporal Worker containers' logs via Docker's
   log driver (e.g. `container` input type, or a bind-mounted `/var/lib/docker/containers`), with
   a conditional processor setting `log_group` to `temporal-server` or `temporal-worker` based on
   the source container's name/label — not two separate Filebeat instances.
3. Point both shippers' `output.logstash.hosts` directly at the local simulated pipeline's
   Logstash port — no tunnel needed, since these run on the same local machine/network as the
   pipeline from Phase 1.
4. Confirm `filebeat test config` passes for both before starting.

### Phase 4 — Confirm shipping

1. Tail each of the **three** shippers' own logs (the GKE DaemonSet, the Colima-k3s DaemonSet,
   and the shared Temporal-stack Filebeat) for "events published" / "Successfully published"
   lines, and confirm no `connection refused` or handshake errors, per shipper.

### Phase 5 — Confirm parsing and search, per log group

1. Open the local Kibana instance and, for **each** `geap_log_group` (`sample-go-service`,
   `temporal-server`, `temporal-worker`, `crossplane-core` — four log groups produced by three
   shippers), confirm:
   - Log entries are present and timestamped correctly.
   - That log shape's meaningful fields are intact and individually filterable — not collapsed
     into a single `message` string. What "meaningful fields" means differs per shape: ADR-007's
     `level`/`request_id`/`user_id` for the sample workload; whatever Temporal Server's own
     structured fields turn out to be (e.g. `service`, `shard-id`); the Temporal Worker's
     workflow/activity/task identifiers; Crossplane's `controller`/reconciled-object identifiers.
   - `geap_log_group`, `client_id`, and tags are present and correctly attributed, and that the
     four log groups are cleanly distinguishable from each other in Kibana.
2. If fields are missing or mis-parsed **for a specific log group**, adjust that log group's
   Logstash filter (GROK/JSON codec) and repeat for that group only — a working filter for one
   log group is not expected to work for another, and iterating per group is itself part of what
   the PoC is measuring, not a failure condition.

### Phase 6 — Record findings

1. Record, **per component/log group**: the working Filebeat config (including the
   metadata-based `log_group` routing rule for the two DaemonSets and the shared Temporal-stack
   shipper), the working Logstash filter config, any parsing iterations needed to get from a
   naive first pass to correctly-parsed fields, whether JSON encoding needed to be explicitly
   configured (Temporal Server, Crossplane), any multiline/stack-trace handling needed, any
   RBAC/hostPath setup the DaemonSets required, and any shipper-version or protocol quirks
   observed.
2. Explicitly record, in `poc-report.md`, that Kerberos/SASL authentication, TLS, real EaaS
   gateway ACLs, and GCP Cloud Interconnect connectivity were **not** exercised — so the report
   cannot be read as validating the network side of Option B — and that Temporal/Crossplane ran
   in a single-cluster, no-real-provider configuration, not their full production topology.

---

## Success Criteria

| Criterion | Pass condition |
|-----------|---------------|
| Simulated EaaS pipeline running | Logstash, Kafka, Elasticsearch, and Kibana all healthy in the local docker-compose stack |
| GKE DaemonSet reachable | The Filebeat DaemonSet in the sandbox GKE cluster successfully connects to the tunnel-exposed Logstash beats input, with no connection/handshake errors |
| Temporal Server + Worker + Crossplane running | All three stood up in Colima (reusing the existing PoC patterns), producing a steady stream of real log lines |
| Colima shippers reachable | Both Colima-hosted shippers (the k3s DaemonSet for Crossplane, the shared Filebeat for the Temporal docker-compose stack) successfully connect directly to the local Logstash beats input, with no connection/handshake errors |
| No sidecar-per-pod/per-container shippers used | Confirmed in `implementation.md`: three shipper instances total (one GKE DaemonSet, one Colima-k3s DaemonSet, one shared Filebeat for the Temporal stack) cover four log groups — not four separate Filebeat instances |
| Logs shipped, all sources | Every shipper's logs show published events with no persistent errors |
| Fields correctly parsed, all four log groups | Each of `sample-go-service`, `temporal-server`, `temporal-worker`, and `crossplane-core` has its meaningful fields individually queryable in Kibana, not flattened into an opaque `message` string — using whatever field shape is correct for that log group, not a single shared schema |
| Metadata-based routing correct | `geap_log_group` is correctly assigned **per entry** from pod/container metadata (not hardcoded per shipper), `client_id` and tags are present, and all four log groups are cleanly distinguishable from each other in Kibana despite sharing shippers |
| JSON-encoding findings recorded | Whether Temporal Server and Crossplane needed explicit config changes to emit JSON logs is recorded, with the actual config used |
| Scope boundary explicit | `poc-report.md` explicitly states that network connectivity, Kerberos/SASL auth, TLS, Temporal multi-cluster replication, and real Crossplane provider reconciliation were not exercised — this PoC proves shipper/parsing correctness per log shape only |

---

## Risks

| Risk | Mitigation / fallback |
|------|---------------------|
| A public tunnel (`cloudflared`/`ngrok`) is itself a PoC-only artifact with no equivalent in the real EaaS architecture | Only synthetic, non-sensitive log data is shipped through it; record it explicitly as a stand-in in `implementation.md`, not a proposed production pattern |
| TCP tunnel behavior (latency, keepalive, reconnect handling) may differ from a real EaaS gateway/Cloud-Interconnect path | Do not use this PoC's tunnel behavior as evidence about real network performance — that question stays with the EaaS team, per the parent research's Open Questions |
| Filebeat version drift — EaaS's documented versions (7.5.x Normal / 7.13.1 Secured) are old; a modern Filebeat may behave differently against an old-style Logstash `beats` input | Pin Filebeat and Logstash's `beats` input plugin to versions close to EaaS's documented ones where practical; record actual versions used in `implementation.md` |
| GROK/JSON parsing config takes multiple iterations to get right | Expected and part of the measurement — record iteration count and final working config, not just the end state |
| Local Elasticsearch/Kibana run with security disabled, unlike either real EaaS tier | Explicitly out of scope; do not present this PoC as validating Secured-tier-equivalent security posture |
| Kafka runs unauthenticated in the stand-in, unlike real EaaS's SASL/Kerberos-gated brokers | Known, deliberate simulation gap — flagged in Scope and must be restated in `poc-report.md`, not silently dropped |
| Temporal Server's and Crossplane's actual default log encoding (JSON vs. text/console) is not confirmed ahead of time — this design assumes it may need explicit configuration but does not assert the exact flag/field name | Confirm the actual config during Phase 3b and record it in `implementation.md`; if a version doesn't support JSON encoding at all, record that as a real finding for Option B's viability for that component, not a PoC blocker to work around |
| Temporal Worker/SDK log lines may include multi-line content (stack traces on activity failures) that a naive single-line Filebeat input would split incorrectly | Configure Filebeat's `multiline` settings for the worker's log group, following the same pattern EaaS's own Filebeat examples use for multi-line `stderr.json` sources (see [evidence.md](evidence.md)) |
| Running Temporal Server + Worker + Crossplane in the same `ucp-crossplane` Colima VM as other PoCs may hit resource contention or port conflicts with those PoCs' own setups | Check for conflicts before starting Phase 3b; use a dedicated Colima profile instead of `ucp-crossplane` if contention appears, and record which profile was actually used in `implementation.md` |
| Crossplane's reconcile-loop logs from a dummy/no-op XRD may be too sparse or too repetitive to represent real provider-reconciliation log volume/shape | Acceptable for this PoC — the goal is confirming the *shape* of controller-runtime's log output is parseable, not measuring realistic reconcile-log volume |
| A Filebeat DaemonSet needs `hostPath`/node-level log access and RBAC for `add_kubernetes_metadata` (watching pods/namespaces cluster-wide) — more privileged than a sidecar reading its own pod's logs | Budget time for RBAC/permission troubleshooting on both the GKE and Colima-k3s DaemonSets; record the actual `ClusterRole`/volume mounts used in `implementation.md`, mirroring how the MonaaS PoC recorded its own RBAC setup as a real component/manifest cost |
| Metadata-based `log_group` routing (one conditional processor covering multiple namespaces/labels) is more complex to configure correctly than a static per-shipper value, and a misconfigured rule could silently misroute one component's logs into another's log group | Verify routing explicitly per log group in Phase 5, not just that *some* data arrived — a wrong `log_group` assignment is a routing bug, not a Logstash parsing issue, and should be diagnosed as such |
| The Temporal docker-compose stack's shared Filebeat depends on being able to read Docker's container logs directly (log driver access), which behaves differently across Docker Desktop/Colima VM configurations | Confirm the actual mechanism that works in the target Colima VM during Phase 3c and record it in `implementation.md`; fall back to bind-mounting each container's log file individually (still one shared Filebeat, just multiple explicit input paths) if the `container` input type doesn't work cleanly |

---

## Sandbox cluster and local environment

| | |
|---|---|
| **Project** | `sub-gcp-ucp-clsd-sandbox` |
| **Cluster** | `ucp-agent-cluster` (region `asia-northeast1`) |
| **Node pool for sample workload + Filebeat** | `system-pool` — no dedicated scale-up expected; this PoC's GKE footprint (one sample workload pod + one Filebeat sidecar) is small relative to the metrics PoCs' component counts |
| **Local device** | Whichever machine runs the docker-compose stand-in pipeline and the outbound tunnel — must stay running for the duration of Phases 2–5 |
| **Colima environment for Temporal + Crossplane** | Reuses the `ucp-crossplane` Colima profile already used by the [Temporal Multi-Cluster Replication PoC](../temporal-multi-cluster-replication/implementation.md) and the [Crossplane XRD Catalog Source PoC](../crossplane-xrd-catalog-source/implementation.md) — Crossplane control plane already installed there; Temporal Server stood up fresh (single cluster, not the source PoC's two-cluster replication setup) via its existing `docker-compose.yml`/`config_template.yaml` |
| **Confirmed reuse note** | Neither Temporal nor Crossplane is deployed on `ucp-agent-cluster` (GKE) — confirmed before writing this design. Both are simulated entirely in Colima, consistent with every other Temporal/Crossplane PoC in this repository, none of which run on GKE either |

---

## Open Questions

- Should this PoC also run a Vector variant shipping directly to the stand-in Kafka (mirroring
  the GCP-C/Session-Hint-Cookie pattern from the parent research), or is validating the
  Filebeat/Logstash path — EaaS's currently-documented default — sufficient on its own? Treated
  as a stretch goal, not required for success criteria.
- If the Filebeat/Logstash-gateway parsing config developed here turns out to differ materially
  from what the real EaaS team would configure on their side during actual onboarding (since
  EaaS's own team, not the tenant, operates the Logstash gateway), how much of this PoC's
  Logstash filter work is directly reusable versus illustrative only? Worth clarifying with the
  EaaS team alongside the network-connectivity confirmation already tracked in the parent
  research.
- Does Temporal Server's actual deployed version (matching the `temporalio/auto-setup:1.25.2`
  image used elsewhere in this repo) expose a documented JSON-logging config field, and does the
  Crossplane version already running in `ucp-crossplane` support a JSON-encoder flag for its
  zap logger? Neither is confirmed as of writing this design — Phase 3b is where this gets
  answered, not assumed here.
- Crossplane **providers** (as opposed to Crossplane core) were explicitly descoped from this
  PoC (see [Scope](#scope)) since a dummy/no-op XRD is enough to exercise core's reconcile-loop
  log shape. If provider-level log output (e.g. `upjet-gcp`'s reconcile logs for a real managed
  resource) turns out to differ meaningfully from core's, is a follow-up PoC needed, or is
  core's shape representative enough for MCUCP-258's purposes? Not answered here.
- The Temporal Worker's SDK-originated log lines (as opposed to UCP's own log statements inside
  the worker binary) depend on which `log.Logger` adapter UCP ultimately wires into
  `worker.Options.Logger` in production — this PoC's worker (adapted from the source PoC) may use
  a different adapter than UCP's real Provisioning/Drift workers will. Confirm with whoever owns
  the Temporal Worker implementation whether the adapter used here is representative.
