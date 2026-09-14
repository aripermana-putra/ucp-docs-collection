---
title: "Crossplane XRD as Catalog Source — PoC Report"
space: UCP
parent_page_id: "../crossplane-xrd-catalog-source.md"
---

# Crossplane XRD as Catalog Source — PoC Report

Answers the research question in [design.md](design.md):

> Can api-server derive catalog entries from Crossplane `CompositeResourceDefinition`
> metadata, persist that derived model in memory and in the Platform DB, and serve it
> through the catalog's existing `List`/`ResolveAvailability` contract — without a live
> cross-cluster call on the request path, and without requiring all 12 real entries to
> be authored first?

## Verdict

Yes. A single dummy XRD carrying the `catalog.ucp.io/*` annotation convention is
successfully derived into a `CatalogItem`, cached in memory, and persisted to the Platform
DB by a process running outside the Crossplane cluster, using the same kubeconfig-based
cross-cluster pattern Argo CD and Cluster API use. The `GET /catalog` and `GET /health/ready`
handlers never call Kubernetes or the Platform DB on the request path — both read only the
in-memory cache. All 8 success criteria from design.md pass. Full proof is in
[implementation.md](implementation.md).

## What Was Proven

| Success criterion (design.md) | Result |
|---|---|
| Cross-cluster read | `xrdclient.Client`, built from a standalone kubeconfig (not `rest.InClusterConfig()`), lists XRDs on `ucp-crossplane` from a process running in a separate cluster/VM |
| Annotation parsing | The dummy XRD's `catalog.ucp.io/*` annotations map correctly to `CatalogItem` fields |
| Derived persistence | The Platform DB row holds the derived `CatalogItem` JSON, not a raw XRD dump |
| Cache-only reads | `GET /catalog` and `GET /health/ready` only ever read `catalog.Cache` |
| DB-seeded warm start | A pod started while Crossplane is unreachable reports `/health/ready` (200) from the last DB-persisted snapshot without blocking on a live poll |
| Reconciliation | Once Crossplane becomes reachable again, the next successful live poll replaces the DB-seeded snapshot — verified by mutating the XRD's annotation post-recovery and observing `/catalog` reflect it without a process restart |
| Stale-on-failure | A poll failure after a successful warm-up keeps serving the last-good snapshot, not an error or an empty list — verified across repeated failures during a real Colima VM outage |
| Cold start | A pod started against an empty Platform DB reports `/health/ready` immediately with an empty catalog, and populates it on the first successful live poll |

## What Was Not Proven

- **Authoring all 12 real catalog entries** — explicitly out of scope; only one dummy XRD
  was used.
- **MCUCP-146's real `gcp-cloud-sql` XRD** — the annotation convention is a draft, not yet
  applied to a real provisioning XRD.
- **Watch/informer-based caching, cache invalidation on individual XRD changes** — out of
  scope; the derived model only updates on the next fixed-interval poll.
- **Multiple api-server pods writing concurrently** — the idempotent-upsert argument in
  design.md was not exercised with more than one poller instance.
- **Deterministic failure timing** — failed polls took up to ~34s against a 5s configured
  interval, relying on the OS TCP connect timeout rather than a bounded per-poll timeout. Does
  not affect correctness (cache-only reads stayed unaffected) but affects how quickly a real
  outage is reflected in logs/metrics.
- **Credential lifecycle** — kubeconfig secret rotation/expiry is explicitly out of scope per
  design.md.

## Recommendation

Adopt the derived-model, DB-seeded approach for the real service catalog. All three follow-ups
below have since been adopted by MCUCP-144's TRD
(`ucp-platform/docs/prd/PRD-005-service-catalog/MCUCP-144-browse-service-catalog.md`), which
builds the poller/cache described here as production code rather than a PoC:

1. Finalize the annotation/label keys and required-vs-optional fields (open in design.md) —
   MCUCP-144 extended the convention with `catalog.ucp.io/roc-subscription-name`. Still open for
   MCUCP-146's real `gcp-cloud-sql` XRD, which has not yet been updated to carry it.
2. Add a bounded per-poll timeout (e.g. `context.WithTimeout` around each `LIST` call) so a
   poll failure is detected in a fixed window rather than depending on OS timeouts —
   MCUCP-144 bounds every `LIST` at a configurable `POLL_TIMEOUT` (default `5s`).
3. Choose a poll interval for production based on expected XRD change frequency vs.
   acceptable staleness (not addressed by this PoC) — MCUCP-144 chose a `24h` steady-state
   interval (configurable), decoupled from a short failure-backoff schedule and an explicit
   refresh endpoint for on-demand propagation, since this PoC's single fixed-interval ticker
   doesn't hold up well at hours/day-scale intervals.

## References

- [design.md](design.md) — scope, hypothesis, approach, success criteria
- [implementation.md](implementation.md) — full verification detail and commands
- [MCUCP-144](https://jira.rakuten-it.com/jira/browse/MCUCP-144) — Browse Service Catalog
- [MCUCP-146](https://jira.rakuten-it.com/jira/browse/MCUCP-146) — Provision Catalog Entry (defines the `CatalogRegistry` interface)
