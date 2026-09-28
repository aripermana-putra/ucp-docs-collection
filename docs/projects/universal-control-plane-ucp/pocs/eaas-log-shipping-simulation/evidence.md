---
title: "EaaS Log Shipping Simulation — Evidence"
space: UCP
parent_page_id: "../eaas-log-shipping-simulation.md"
---

# EaaS Log Shipping Simulation — Evidence

Supporting evidence doc. References used during setup and troubleshooting.

- [Filebeat `add_kubernetes_metadata` processor
  reference](https://www.elastic.co/guide/en/beats/filebeat/7.17/add-kubernetes-metadata.html) —
  matchers, `logs_path`, and the RBAC (`namespaces`/`pods`/`nodes` `get`/`list`/`watch`) it needs.
- [Filebeat `container` input
  reference](https://www.elastic.co/guide/en/beats/filebeat/7.17/filebeat-input-container.html) —
  confirms it reads every matching container log file on the node by default, the root cause of
  the unscoped-harvesting bug found in Phase 3 (see
  [implementation.md](implementation.md#phase-3--sample-workload--filebeat-daemonset-gke)).
- [Filebeat `add_fields` processor
  reference](https://www.elastic.co/guide/en/beats/filebeat/7.17/add-fields.html) — the `target`
  option's behavior with dotted-string vs. nested-YAML targets, relevant to the
  `fields.geap`-nesting bug found in the same phase.
- [Logstash `beats` input plugin](https://www.elastic.co/guide/en/logstash/7.17/plugins-inputs-beats.html)
  and [`kafka` output plugin](https://www.elastic.co/guide/en/logstash/7.17/plugins-outputs-kafka.html) —
  used for the stand-in pipeline's input/durability-buffer stages.
- [Logstash `json` filter](https://www.elastic.co/guide/en/logstash/7.17/plugins-filters-json.html)
  (`skip_on_invalid_json`) — used for the `crossplane-core` filter's mixed JSON/plain-text
  handling.
- [Apache Kafka KRaft mode
  configuration](https://kafka.apache.org/37/documentation.html#kraft) — used for the stand-in
  pipeline's single-broker Kafka (no Zookeeper), after `bitnami/kafka:3.7` was found retired
  (`docker.io/bitnami/kafka:3.7: not found`) and replaced with `apache/kafka:3.7.0`.
- [ngrok TCP tunnels documentation](https://ngrok.com/docs/universal-gateway/tcp/) — confirms an
  authtoken/account is required to open a TCP tunnel; this PoC hit `ERR_NGROK_4018` with no
  account available.
- [Cloudflare Tunnel — TCP
  applications](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/private-net/cloudflared/)
  and [`cloudflared access
  tcp`](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tcp/) —
  the documented client-side mechanism for reaching a TCP tunnel; this PoC's anonymous
  "quick tunnel" + `access tcp` combination failed with `websocket: bad handshake`, not
  independently root-caused against a real Cloudflare account.
- [Temporal Server configuration
  reference](https://docs.temporal.io/references/configuration) — `log.encoding`, confirmed
  unnecessary to set explicitly since the deployed `temporalio/auto-setup`-based server already
  emits JSON by default (see [implementation.md's Phase
  3b](implementation.md#phase-3b--temporal-server-temporal-worker-crossplane-core-colima)).
- [Temporal Go SDK — `worker.Options.Logger` /
  `log.Logger`](https://pkg.go.dev/go.temporal.io/sdk/log) — the pluggable logger interface whose
  *default* implementation was confirmed (by direct observation, not by reading this doc alone)
  to produce plain-text, not JSON, output for `ucp-provisioning-worker`/`drift-worker`.
- [Crossplane — logging](https://docs.crossplane.io/latest/guides/troubleshoot-crossplane/) —
  general Crossplane operability reference consulted while investigating whether an explicit
  JSON-encoder flag was needed; not needed in practice, since the deployed Crossplane 2.2.0
  already emits zap JSON by default.
