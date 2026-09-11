---
title: "Crossplane XRD Versioned Schema and Provisioning — Implementation"
space: UCP
parent_page_id: "../crossplane-xrd-versioned-schema-provisioning.md"
---

# Crossplane XRD Versioned Schema and Provisioning — Implementation

Supporting proof document. Describes what was built, run, and verified for Phases 1-5 of
[design.md](design.md).

## Source Code

| | |
|---|---|
| **Repository** | [`aripermana-putra/kitchen-sink`](https://github.com/aripermana-putra/kitchen-sink) |
| **Path** | `crossplane-xrd-versioned-schema-provisioning-poc/` |
| **Entry point** | `cmd/apiserver/main.go` (`func main`) |
| **Commit** | [`b7c6a88`](https://github.com/aripermana-putra/kitchen-sink/commit/b7c6a88) |

Module: `github.com/aripermana-putra/kitchen-sink/crossplane-xrd-versioned-schema-provisioning-poc`
(Go 1.26.1).

| File | Role |
|---|---|
| `crossplane/xrd/gcp-gke.xrd.yaml` | Multi-version dummy XRD (`v1alpha1`, `v1beta1`) |
| `internal/catalog/types.go` | `Item`, `EntrySchema` (`Versions map[string]map[string]any`, `StorageVersion`, `Group`, `Kind`, `Resource`), `ResourceGraphNode` |
| `internal/catalog/catalog.go` | `Cache` — `sync.RWMutex`-guarded, full-slice-replace `Swap`, `GetEntrySchema` |
| `internal/catalog/poller.go` | `Poller` — immediate first poll, then fixed-interval, stale-on-failure |
| `internal/xrdclient/xrdclient.go` | `Client.ListCatalogItems` — derives `Item` and `EntrySchema` (including `Group`/`Kind`/`Resource` from `spec.group`/`spec.names`) per XRD in one `LIST` |
| `internal/validate/validate.go` | `Against` — compiles and validates a JSON Schema map against an object via `santhosh-tekuri/jsonschema/v6` |
| `internal/httpapi/httpapi.go` | `Server` — `handleDescribe`, `handleTemplate`, `handleProvision`, `handleReady`; `handleProvision` builds the applied XR's GVR from `EntrySchema.Group`/`Kind`/`Resource`, no static target registry |
| `cmd/apiserver/main.go` | Wires kubeconfig-based `dynamic.Interface`, `xrdclient`, `Poller`, `httpapi.Server` |

## Environment

Local Crossplane cluster via a dedicated Colima VM (`ucp-crossplane` profile), kubeconfig at
`kubeconfig-crossplane.yaml` (regenerated after a VM recreate — see Risks below). No availability
service, no real cloud backend, no `Composition` bound to the XRD.

## What Was Built

### Phase 1 — Multi-version dummy XRD

`crossplane/xrd/gcp-gke.xrd.yaml`: `apiextensions.crossplane.io/v2`, `scope: Namespaced`,
`kind: XGCPGKECluster`, group `gcp-gke.catalog.ucp.io`. Two `served: true` versions:

- `v1alpha1` — `shared` (`projectId` required, `location`), `cluster.releaseChannel`,
  `nodepool.nodeCount`.
- `v1beta1` — adds `cluster.network` and `cluster.subnetwork` on top of `v1alpha1`.

Carries the same `catalog.ucp.io/*` annotations as the prior PoC (`service-id`, `name`,
`provider`, `category`, `description`, `resource-graph`, `supported-regions`), plus the
`resource-graph` JSON: `cluster` → `nodepool` (depends on `cluster`) → `conn-secret` (depends on
`cluster`).

**Finding — `storage: true` is not a valid XRD field.** Design.md's Phase 1/Phase 6 wording
(carried over from MCUCP-145's TRD's literal worked example) describes marking a version
`storage: true`/`storage: false` directly on `spec.versions[]`. Applying that YAML failed with
`strict decoding error: unknown field "spec.versions[0].storage"`. `kubectl explain
compositeresourcedefinition.spec.versions` confirms Crossplane v2 XRDs use `referenceable:
true/false` instead — "Exactly one version must be marked as referenceable... It's mapped to the
CRD's `spec.versions[*].storage` field." The XRD was corrected to use `referenceable: false`
(`v1alpha1`) / `referenceable: true` (`v1beta1`); verified post-apply that the generated CRD's
`spec.versions[*].storage` is set correctly (`false true`) via `kubectl get crd ... -o
jsonpath='{.spec.versions[*].storage}'`. This is a real discrepancy between MCUCP-145's TRD's
literal YAML and Crossplane v2's actual API surface — the TRD's *model* (one version is "the"
storage/served-as-canonical version) holds, only the field name it writes is wrong.

### Phase 2 — Schema-deriving poller

`xrdclient.deriveEntrySchema` reads `spec.versions[]`, keeps only `served: true` entries, copies
`schema.openAPIV3Schema.properties.spec.properties.parameters` verbatim per version into
`EntrySchema.Versions[name]`, and sets `StorageVersion` to whichever version has
`referenceable: true`. Reuses the prior PoC's `Cache`/`Poller` pattern unchanged (immediate first
poll, fixed-interval ticker, stale-on-failure).

### Phase 3 — Describe and template endpoints

`GET /catalog/{serviceId}` returns `Versions[StorageVersion]` only, alongside the resource graph
and `storageVersion` name. `GET /catalog/{serviceId}/template` renders the same schema as YAML,
matching MCUCP-145's worked `--generate-template` example exactly: a fixed
`schemaVersion: <StorageVersion>` line, a `name: ""` placeholder, then `shared` (if present)
followed by one group per resource-graph item in graph order (properties sorted alphabetically
within each group), separated by a blank line. Each property gets a
`# <name> (required|optional) — <description> (default: <default>)` comment, a second
`#   example: <value>` comment line when the schema has one, and its value is always emitted
`""` regardless of any default — nothing is pre-filled, matching the TRD's "no validation at
generation time" rule.

### Phase 4 — Provision endpoint

`POST /catalog/{serviceId}/provision` decodes `{name, schemaVersion, parameters}`, looks up
`Versions[schemaVersion]` (not `StorageVersion`), 404s if that version isn't served/stored.
Validates `parameters` against that exact version's schema via `validate.Against`. On failure,
returns 400 with the validator's error and does not call `Create`. On success, builds an
`unstructured.Unstructured` (`apiVersion: <group>/<schemaVersion>`, `kind`, `metadata.name`,
`spec.parameters`) and applies it via the dynamic client's `Create` at
`namespace: default` — no Claim.

## Verification (Phase 5)

All four Phase 5 checks were run against the live cluster with the `apiserver` binary started via
`go run ./cmd/apiserver` (kubeconfig pointed at the recreated `ucp-crossplane` VM) and confirmed
independently via `kubectl`. Exact request/response payloads and `kubectl` output are recorded in
[poc-report.md](poc-report.md) (test data belongs there, not here).

1. `GET /catalog/gcp-gke` and `GET /catalog/gcp-gke/template` — confirmed `storageVersion:
   v1beta1` and that `cluster.network`/`cluster.subnetwork` (v1beta1-only fields) are present;
   `v1alpha1` is never surfaced by either endpoint.
2. Filled the `v1beta1` template, submitted via `provision` with `schemaVersion: v1beta1` — 201
   Created; confirmed via `kubectl get xgcpgkeclusters` that the XR exists with `SYNCED: False`
   (no `Composition` bound, as designed).
3. Submitted a payload omitting the required `shared.projectId` field — 400 with the validator's
   `missing property 'projectId'` error; confirmed via `kubectl get xgcpgkeclusters
   <name>` returning `NotFound` — no XR was created.
4. Submitted a payload tagged `schemaVersion: v1alpha1` (only `shared`, `cluster.releaseChannel`,
   `nodepool.nodeCount` — no `network`/`subnetwork`) — 201 Created; fetched it back via `kubectl
   get --raw ".../apis/gcp-gke.catalog.ucp.io/v1alpha1/namespaces/default/xgcpgkeclusters/<name>"`
   and confirmed the API server serves it back as `apiVersion: v1alpha1` even though `v1beta1` is
   the storage version — Kubernetes' built-in multi-version round-trip (no conversion webhook
   needed, since `v1beta1` is a structurally additive superset of `v1alpha1`) is what makes the
   "old version stays valid" claim hold.

All Success Criteria rows in design.md covering Phases 1-5 pass.

## Phase 6 — Zero-code-change extensibility

Two changes were applied to the live cluster, then the Crossplane pod was restarted and the
PoC's own poller was left to pick them up on its normal fixed interval — no manual trigger,
no restart of the PoC process for the schema-derivation path:

1. A brand-new dummy XRD, `gcp-cloud-sql` (single version `v1alpha1`, `instance`/`conn-secret`
   resource graph, `XGCPCloudSQLInstance` kind) — a different `serviceId` entirely, unrelated to
   `gcp-gke`.
2. A third version, `v1gamma1`, added to the existing `gcp-gke` XRD (adds
   `nodepool.maxNodeCount` on top of `v1beta1`), marked `referenceable: true`; `v1beta1` demoted
   to `referenceable: false` (`v1alpha1` unchanged).

`kubectl delete pod` on the running `crossplane` pod, followed by `kubectl wait
--for=condition=Ready`, confirmed Crossplane accepted both CRD changes cleanly on restart — no
crash loop, no stale state (this time; contrast with the earlier VM-recreate crash loop, which
was caused by a stale *data disk*, not an XRD/CRD change). Post-restart, `kubectl get crd
xgcpgkeclusters... -o jsonpath='{.spec.versions[*].storage}'` confirmed `false false true`,
i.e. `v1gamma1` is now the CRD's storage version.

After the PoC's poller ran its next scheduled poll (no manual trigger), both endpoints picked up
the changes with **no changes to `internal/catalog`, `internal/xrdclient`, `internal/httpapi`, or
`internal/validate`**:

- `GET /catalog/gcp-gke` — `storageVersion` is now `v1gamma1`; schema includes
  `nodepool.maxNodeCount`.
- `GET /catalog/gcp-cloud-sql` — fully describable on first poll after apply, with no prior
  knowledge of this `serviceId` anywhere in the code.
- `POST /catalog/gcp-gke/provision` — `schemaVersion: v1gamma1`, `v1beta1`, and `v1alpha1` **all
  three** validated and applied successfully in the same run, proving old versions keep working
  across more than one version bump, not just the immediately-previous one.

**Scoped exception — provision's XR-target lookup — resolved.** `main.go`'s `xrTargets` map
(`Group`, `Kind`, `Resource` per `serviceId`) was originally a static registry, deliberately out
of scope per design.md ("XR-target derivation" excluded from Scope). It has since been removed:
`catalog.EntrySchema` now carries `Group`/`Kind`/`Resource`, read via three `unstructured.NestedString`
calls (`spec.group`, `spec.names.kind`, `spec.names.plural`) in the same `deriveEntrySchema` call
that already reads `spec.versions[]` — no second call, no separate `XRTarget` type.
`httpapi.NewServer` no longer takes an `xrTargets` argument; `handleProvision` builds the applied
XR's `apiVersion`/`kind` and the dynamic client's `GroupVersionResource` from the cached
`EntrySchema` (`es.Group`/`es.Kind`/`es.Resource`) directly. Re-verified against the live cluster
after the change: `POST /catalog/gcp-gke/provision` (`v1gamma1`) and
`POST /catalog/gcp-cloud-sql/provision` (`v1alpha1`) both return `201 Created` with zero entries
in any static map, confirmed via `kubectl get xgcpgkeclusters`/`xgcpcloudsqlinstances`.

`describe`/`template`/schema derivation and both parts of provision — *validation* and *target
resolution* — are now all confirmed zero-code-change.

## Risks Observed

| Risk (from design.md) | Observed |
|---|---|
| Crossplane rejects applying an XR with no `Composition` bound | Not observed — `Create` succeeds; the XR sits with `SYNCED: False` and a `cannot select Composition: no compatible Compositions found` status condition, matching the design's expectation exactly |
| K8s structural schema vs. the JSON Schema validator library diverge | Not observed — both a passing and a deliberately failing payload behaved as expected |
