---
title: "Multi-Cluster ProviderConfig Sync PoC — Report"
space: UCP
parent_page_id: "../multi-cluster-provider-config-sync.md"
---

# Multi-Cluster ProviderConfig Sync PoC — Report

| | |
|---|---|
| **Ticket** | [MCUCP-306](https://jira.rakuten-it.com/jira/browse/MCUCP-306) |
| **Status** | Complete |
| **Research question** | Does a GitOps pull model (ArgoCD `ApplicationSet` matrix generator) correctly and automatically extend `ExternalSecret` + `ProviderConfig` pairs to a newly added Crossplane cluster, without a new commit or a new UCP backend code path? |

---

## Verdict

**Confirmed.** A matrix `ApplicationSet` (cluster generator × tenant registry
generator), rendering a single shared Helm chart, backfills an existing
provider-config registration onto a newly registered cluster automatically. The
only actions required when a cluster is added are registering it with ArgoCD and
running the platform bootstrap step (ESO + `ClusterSecretStore` install) — no
registry change, no new commit, no UCP backend involvement.

---

## What This PoC Answers

The [parent research doc](../../research/multi-cluster-provider-config-sync.md)
identified GitOps pull (ArgoCD) as the preferred mechanism for keeping
`ExternalSecret` + `ProviderConfig` pairs correct across a growing set of
Crossplane clusters, over a custom Temporal-based reconciler. Before committing to
that path, the following needed hands-on proof:

- Whether the matrix `ApplicationSet` pattern actually backfills an existing
  tenant registration onto a newly added cluster with zero manual intervention
- Whether GCP Secret Manager, ESO, and Crossplane's `ProviderConfig` chain
  together correctly end-to-end, not just in isolation
- What operational gaps this mechanism has in practice (registry granularity,
  cluster-scoping the generator, credential handling for ESO itself)

---

## What Was Proved

- The matrix `ApplicationSet` pattern works as designed: adding a second cluster
  to ArgoCD's registered-cluster list, with no Git commit and no registry change,
  produces a new `Application` for every existing tenant registration on that
  cluster within one reconcile pass.
- The full credential path works end-to-end with the real credential shape: a GCP
  Secret Manager secret holding the actual `external_account` credential config
  JSON (the same shape validated in the [wif-gcp PoC](../wif-gcp/poc-report.md)),
  pulled whole by ESO into a K8s `Secret` via a `ClusterSecretStore`, referenced
  by a `ProviderConfig` that reports `Healthy` — on two independent clusters, from
  one registry entry. The registry entry itself carries no copy of anything
  inside that payload — only `projectID`, which `ProviderConfig.spec` requires as
  a literal field regardless of credential source.
- The tenant registry's per-registration granularity (not per-tenant) works as
  designed — see the [multi-project-gcp PoC](../multi-project-gcp.md) for why that
  granularity was chosen.
- Scoping the cluster generator by label (`mcucp.io/role: crossplane`) correctly
  excludes ArgoCD's own built-in `in-cluster` entry from the fan-out, keeping the
  `Application` list limited to real target clusters.
- Cluster decommission cleanup: deleting a cluster's ArgoCD cluster `Secret`
  triggers the matrix `ApplicationSet` controller to automatically delete every
  `Application` it had generated for that cluster within one reconcile pass
  (observed within ~20 seconds) — no manual `argocd app delete` step needed.
  This cleanup is scoped to ArgoCD's own bookkeeping only: resources already
  applied to the target cluster (`ExternalSecret`, `ProviderConfig`,
  `ClusterSecretStore`, the ESO deployment) are left running, unmanaged, since
  ArgoCD no longer has credentials to reach that cluster once its `Secret` is
  removed.

---

## What Was Not Proved

- ESO's own GCP Secret Manager credential path via Workload Identity — these are
  local, non-GKE clusters, so ESO used a stored service account key instead (see
  [implementation.md](./implementation.md), "ESO's own GCP Secret Manager credential").
  Production WIF-based ESO auth is unverified until run against GKE.
- Behavior at representative registration × cluster scale. The research doc's
  scale concern was already flagged as low priority for UCP's internal-IDP
  tenant base; this PoC ran with a single registration and two clusters, not a
  representative count.
- The ROC provider-config path — only GCP was exercised, per the PoC's scope.

---

## Findings Summary

The GitOps pull mechanism resolves the two questions the parent research
document left open for the fan-out problem: *when* a (registration, cluster)
pair gets created (as soon as the matrix generator next reconciles after either
side changes) and *how* the set stays correct as clusters change (automatically
— the mechanism that handles cluster 2 appearing is the same one that rendered
cluster 1, with no dedicated backfill code path).

The credential chain — GCP Secret Manager as the single source of truth, ESO as
the per-cluster sync bridge, `ProviderConfig` as the Crossplane-facing consumer —
works exactly as the reference architecture in the research doc describes, with
one gap: ESO's own authentication to GCP Secret Manager needs the same
credential-mechanism decision that Crossplane's provider pods already went
through in the [wif-gcp PoC](../wif-gcp/poc-report.md) — WIF/Workload Identity on
GKE, a stored key everywhere else.

---

## Recommendation

**Adopt the GitOps pull model (ArgoCD `ApplicationSet` matrix generator) for
multi-cluster `ProviderConfig` sync.** Proceed with the following before
production adoption:

1. **ESO Workload Identity on GKE** must be validated the same way the provider
   pod's WIF path was validated in the wif-gcp PoC, before relying on it for
   ESO's own Secret Manager access in production.
2. **Cluster decommission runbook** — ArgoCD's own bookkeeping self-heals (see
   above), so no cleanup step is needed there. A runbook is still needed for the
   case where a cluster is deregistered from ArgoCD but kept running for another
   purpose: its `ExternalSecret`/`ProviderConfig`/`Secret` objects will not be
   pruned automatically and must be cleaned up deliberately if that scenario is
   ever real for UCP.
3. **GCP Secret Manager IAM scoping** for ESO's identity — this PoC granted
   project-wide `secretmanager.secretAccessor`, which is not the production
   posture; the research doc's open question on per-secret conditional IAM vs.
   project-wide access still needs a decision.
4. **Cluster registration automation** — this PoC registered clusters with
   ArgoCD by hand (scripted, but manually triggered). A real Ops cluster
   provisioning pipeline needs this as an automated step, including the
   `mcucp.io/role: crossplane`-equivalent labeling that scopes the generator.

---

## Supporting Detail

- [Design](./design.md) — question, hypothesis, scope, approach, success criteria
- [Implementation](./implementation.md) — environment, GitOps structure, verification results
- Reproduction scripts and full build-time troubleshooting: `multi-cluster-provider-config-sync-poc/`
  in the `kitchen-sink` repo
