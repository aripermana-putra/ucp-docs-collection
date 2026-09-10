---
title: "Crossplane XRD Versioned Schema and Provisioning — PoC Report"
space: UCP
parent_page_id: "../crossplane-xrd-versioned-schema-provisioning.md"
---

# Crossplane XRD Versioned Schema and Provisioning — PoC Report

Human-first verdict document. See [design.md](design.md) for scope and hypothesis,
[implementation.md](implementation.md) for full build/run detail.

## Research Question Answered

> Can api-server derive a per-served-version JSON Schema from a `CompositeResourceDefinition`'s
> own OpenAPI schema, serve it through `describe`/`generate-template`-equivalent reads that only
> ever show the current storage version, and then validate a filled-in submission against the
> exact version it was generated against before applying it as a namespaced Composite Resource —
> without an availability check and without a real provisioning backend?

**Status: Phases 1-6 complete.**

## Verdict

Yes. A multi-version dummy `gcp-gke` XRD (`v1alpha1`, `v1beta1`, later `v1gamma1`) was polled into
a `Versions map[string]map[string]any` keyed by version name plus a `StorageVersion` pointer;
`describe`/`template` endpoints render only `Versions[StorageVersion]`; `provision` validates a
submission against `Versions[schemaVersion]` (the version the caller names, not necessarily the
storage version) and applies it as a Crossplane v2 namespaced XR only on successful validation.
An old version (`v1alpha1`) submitted after `v1beta1` became the storage version still validated
and applied successfully — the core claim MCUCP-145's TRD depends on.

## What Was Proven

- Multi-version schema derivation from a real XRD's `spec.versions[]` into `Versions`/
  `StorageVersion`, reusing the prior PoC's poller pattern with no new caching mechanism.
- `describe` and `template` never surface a non-storage version.
- A submission is validated against the exact `schemaVersion` it declares, not the current
  storage version — and an older version continues to validate and apply successfully after a
  newer version becomes current.
- Invalid input is rejected before any apply call, with no XR left behind.
- Applying an XR with no bound `Composition` succeeds and leaves it in an expected
  unsynced/pending state, not a rejected one.
- A brand-new, previously-unknown XRD (`gcp-cloud-sql`) becomes fully describable and
  templatable with zero changes to the poller or HTTP handlers, after apply + Crossplane
  restart + the PoC's next scheduled poll.
- A new version (`v1gamma1`) added to an already-known XRD becomes the current storage version
  with zero poller/handler code change, and both prior versions (`v1beta1`, `v1alpha1`)
  continue to validate and apply successfully — proving old versions survive more than one
  version bump, not just the immediately-previous one.
- Crossplane restarts cleanly on an XRD/CRD schema change with no crash loop, given a healthy
  underlying data disk (see Finding below for the contrasting case where it wasn't).

## What Was Not Proven

- Whether Crossplane's restart was strictly *necessary* to pick up the CRD change, versus
  Kubernetes' API server serving it immediately on its own — this PoC restarted Crossplane every
  time and did not test the without-restart path, so the restart's necessity remains unconfirmed
  (see design.md's Risks — this was flagged as an open question going in, not resolved by this
  run).
- Zero-code-change extensibility for provision's *XR-target resolution* (the `Group`/`Kind`/
  `Resource` lookup) — this is a known, explicitly scoped-out exception, not something this PoC
  set out to prove (see Finding below).
- Anything about real provisioning, availability checks, or the real MCUCP-146 `provision`
  endpoint's shape — explicitly out of scope (design.md Scope).

## Finding — Resolved

MCUCP-145's TRD's worked `gcp-gke` example wrote `storage: true`/`storage: false` directly on
an XRD's `spec.versions[]` entries. That field does not exist on Crossplane v2
`CompositeResourceDefinition` — applying it fails with `strict decoding error: unknown field
"spec.versions[0].storage"`. The real field is `referenceable: true/false`, which Crossplane
maps internally to the generated CRD's `spec.versions[*].storage`. The TRD's *model* (exactly one
version is "the" current/storage version) is correct and holds up under this PoC; only the field
name in its literal YAML was wrong. See [implementation.md](implementation.md#phase-1--multi-version-dummy-xrd)
for how this was confirmed.

A follow-up read of MCUCP-145's own TRD also found its worked example marking both `v1alpha1` and
`v1beta1` `referenceable: true` simultaneously — invalid, since exactly one version can be
`referenceable: true` at a time. The TRD's worked YAML example and its `EntrySchema`
derivation/implementation prose (`StorageVersion`, `deriveEntrySchema`, `DescribeCatalogEntry`)
have both been corrected to use `referenceable: true` consistently, with only one version marked
`referenceable: true` at a time.

## Finding — Provision's XR-target lookup is a scoped exception, and is likely trivially fixable

Provisioning the new `gcp-cloud-sql` XRD required adding one entry to `main.go`'s static
`xrTargets` map before the running `apiserver` picked it up — the one part of Phase 6 that is
not zero-code-change. This was scoped out deliberately going in (design.md Scope: "XR-target
derivation" is out of scope), not discovered as an unplanned gap. Worth noting for any follow-up:
`Group` (`spec.group`), `Kind` (`spec.names.kind`), and `Resource` (`spec.names.plural`) are all
present verbatim on the same XRD object already being `LIST`ed for schema derivation — reading
them directly, rather than maintaining a static map, looks like a small follow-up, not a hard
problem. Not implemented here since it was out of scope.

## Test Data

### Describe (`GET /catalog/gcp-gke`)

```json
{
  "serviceId": "gcp-gke",
  "resourceGraph": [
    {"name": "cluster", "type": "Cluster"},
    {"name": "nodepool", "type": "NodePool", "dependsOn": ["cluster"]},
    {"name": "conn-secret", "type": "ConnectionSecret", "dependsOn": ["cluster"]}
  ],
  "storageVersion": "v1beta1",
  "parametersSchema": {
    "properties": {
      "cluster": {"properties": {
        "network": {"type": "string", "description": "VPC network name"},
        "releaseChannel": {"type": "string", "default": "REGULAR", "description": "REGULAR | RAPID | STABLE"},
        "subnetwork": {"type": "string", "description": "Subnetwork name"}
      }, "type": "object"},
      "nodepool": {"properties": {"nodeCount": {"type": "integer", "default": 1}}, "type": "object"},
      "shared": {"properties": {
        "location": {"type": "string", "default": "us-central1"},
        "projectId": {"type": "string"}
      }, "required": ["projectId"], "type": "object"}
    },
    "type": "object"
  }
}
```

### Template (`GET /catalog/gcp-gke/template`)

```yaml
# schemaVersion (fixed) — schema version this template was generated against; leave as-is
schemaVersion: v1beta1
# name (required) — resource name
name: ""
cluster:
  # VPC network name
  network: ""
  # REGULAR | RAPID | STABLE
  releaseChannel: REGULAR
  # Subnetwork name
  subnetwork: ""
nodepool:
  # Initial node count
  nodeCount: 1
shared:
  # Region/zone for cluster and node pool
  location: us-central1
  # GCP project ID
  projectId: ""
```

### Provision — valid `v1beta1` submission → 201 Created

Request:

```json
{
  "name": "gke-beta-test",
  "schemaVersion": "v1beta1",
  "parameters": {
    "shared": {"projectId": "coupon-prod-gcp", "location": "us-central1"},
    "cluster": {"releaseChannel": "REGULAR", "network": "default", "subnetwork": "default"},
    "nodepool": {"nodeCount": 3}
  }
}
```

Response (201):

```json
{"apiVersion":"gcp-gke.catalog.ucp.io/v1beta1","kind":"XGCPGKECluster","name":"gke-beta-test","schemaVersion":"v1beta1"}
```

`kubectl get xgcpgkeclusters -n default`:

```
NAME            SYNCED   READY   COMPOSITION   COMPOSITIONREVISION   AGE
gke-beta-test   False                                                13s
```

### Provision — invalid submission (missing required field) → 400, no XR created

Request: same as above with `"shared": {"location": "us-central1"}` (no `projectId`).

Response (400):

```json
{"error":"validation failed","detail":"jsonschema validation failed with 'file:///.../params.json#'\n- at '/shared': missing property 'projectId'"}
```

`kubectl get xgcpgkeclusters gke-invalid-test -n default`:

```
Error from server (NotFound): xgcpgkeclusters.gcp-gke.catalog.ucp.io "gke-invalid-test" not found
```

### Provision — old `v1alpha1` submission after `v1beta1` is current → 201 Created

Request:

```json
{
  "name": "gke-alpha-test",
  "schemaVersion": "v1alpha1",
  "parameters": {
    "shared": {"projectId": "coupon-prod-gcp", "location": "us-central1"},
    "cluster": {"releaseChannel": "REGULAR"},
    "nodepool": {"nodeCount": 1}
  }
}
```

Response (201):

```json
{"apiVersion":"gcp-gke.catalog.ucp.io/v1alpha1","kind":"XGCPGKECluster","name":"gke-alpha-test","schemaVersion":"v1alpha1"}
```

Fetched back at its own version via `kubectl get --raw
"/apis/gcp-gke.catalog.ucp.io/v1alpha1/namespaces/default/xgcpgkeclusters/gke-alpha-test"`:

```json
{
  "apiVersion": "gcp-gke.catalog.ucp.io/v1alpha1",
  "kind": "XGCPGKECluster",
  "metadata": {"name": "gke-alpha-test", "namespace": "default"},
  "spec": {"parameters": {
    "cluster": {"releaseChannel": "REGULAR"},
    "nodepool": {"nodeCount": 1},
    "shared": {"location": "us-central1", "projectId": "coupon-prod-gcp"}
  }},
  "status": {"conditions": [
    {"type": "Synced", "status": "False", "reason": "ReconcileError", "message": "cannot select Composition: no compatible Compositions found"},
    {"type": "Responsive", "status": "True", "reason": "WatchCircuitClosed"}
  ]}
}
```

Confirms Kubernetes' built-in multi-version storage/serving round-trip serves the object back
under the version it was created with (`v1alpha1`), even though it is stored internally under
`v1beta1` — no conversion webhook was needed because `v1beta1` is an additive superset of
`v1alpha1`.

### Phase 6 — new XRD (`gcp-cloud-sql`) describe, zero code change

```json
{
  "serviceId": "gcp-cloud-sql",
  "resourceGraph": [
    {"name": "instance", "type": "Instance"},
    {"name": "conn-secret", "type": "ConnectionSecret", "dependsOn": ["instance"]}
  ],
  "storageVersion": "v1alpha1",
  "parametersSchema": {
    "properties": {
      "instance": {"properties": {
        "databaseVersion": {"type": "string", "default": "POSTGRES_15"},
        "tier": {"type": "string", "default": "db-f1-micro"}
      }, "type": "object"},
      "shared": {"properties": {
        "location": {"type": "string", "default": "us-central1"},
        "projectId": {"type": "string"}
      }, "required": ["projectId"], "type": "object"}
    },
    "type": "object"
  }
}
```

### Phase 6 — `gcp-gke` after adding `v1gamma1`, describe now shows the new storage version

`storageVersion` changed from `v1beta1` to `v1gamma1`; schema now includes
`nodepool.maxNodeCount` (default `5`).

### Phase 6 — all three `gcp-gke` versions provision successfully in one run

| `schemaVersion` | Result |
|---|---|
| `v1gamma1` (current storage version) | 201 Created |
| `v1beta1` (one version behind) | 201 Created |
| `v1alpha1` (two versions behind) | 201 Created |

`kubectl get xgcpgkeclusters -n default`:

```
NAME              SYNCED   READY   COMPOSITION   AGE
gke-alpha-again   False                          100s
gke-alpha-test    False                          13m
gke-beta-again    False                          100s
gke-beta-test     False                          14m
gke-gamma-test    False                          100s
```

### Phase 6 — new XRD provisions successfully after adding one static `xrTargets` entry

```json
{"apiVersion":"gcp-cloud-sql.catalog.ucp.io/v1alpha1","kind":"XGCPCloudSQLInstance","name":"sql-alpha-test","schemaVersion":"v1alpha1"}
```

`kubectl get xgcpcloudsqlinstances -n default`:

```
NAME             SYNCED   READY   COMPOSITION   AGE
sql-alpha-test   False                          26s
```

### Crossplane pod restart — CRD storage version after restart

```
$ kubectl get crd xgcpgkeclusters.gcp-gke.catalog.ucp.io -o jsonpath='{.spec.versions[*].name}{"\n"}{.spec.versions[*].storage}{"\n"}'
v1alpha1 v1beta1 v1gamma1
false false true
```

## Recommendation

The `EntrySchema`/`StorageVersion` model in MCUCP-145's TRD, and the validate-then-apply pattern
MCUCP-146's provisioning is expected to follow, both hold up against a real multi-version XRD, a
real JSON Schema validator, a Crossplane restart, and a second version bump. Recommend:

1. MCUCP-145's TRD has been corrected to use `referenceable: true/false` instead of the
   non-existent `storage: true/false` field (see Finding above) — done.
2. Treating this PoC as closed for its stated research question — all of design.md's success
   criteria are met, with provision's XR-target resolution as the one explicitly scoped
   exception (see Finding above).
