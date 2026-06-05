---
title: "ADR-001: Repository Structure"
space: UCP
parent_page_id: "../implementation.md"
---

# ADR-001: Repository Structure for UCP Platform
**Status:** Proposed
**Date:** 2026-06-05
**Decision makers:** UCP Group

---

## Context and Problem Statement

All UCP components currently live in `ucp-platform` as git submodules pointing to
separate repositories at pinned commit SHAs:

| Component | Type | Role |
|---|---|---|
| `api-server` | Go service (submodule) | REST API backend |
| `temporal-worker` | Go service (submodule) | Workflow engine — binaries: `provisioning-worker`, `drift-worker` |
| `cli` | Go binary (submodule) | Command-line client, generated from OpenAPI spec |
| `crossplane/` | YAML (in-repo) | XRDs, Compositions, ProviderConfigs, example claims |
| `k8s/` | Shell + YAML (in-repo) | Local deployment scripts |
| `provider-roc` | Go module (submodule) | Custom Crossplane provider, published as versioned OCI image |
| `omnia-client` | Go module (submodule) | Generated HTTP client from Omnia OpenAPI spec |
| `frontend` | React app (submodule) | Web dashboard |
| `docs` | Markdown (submodule) | Team knowledge base |

The current structure causes recurring friction: every feature that touches `api-server`,
`temporal-worker`, and `cli` simultaneously requires 3–4 coordinated pull requests across
separate repositories, a submodule pointer update in the umbrella, and manual effort to keep
the OpenAPI spec in sync across repos. The decision is how to reorganize these components
to reduce coordination overhead without sacrificing independent deployment capability.

---

## Decision Drivers

- `api-server`, `temporal-worker`, and `cli` change together on every feature delivery. Adding a new resource type requires changes in all three simultaneously.
- The OpenAPI spec is consumed by both `api-server` (server stubs) and `cli` (HTTP client). These two consumers must always stay in sync with the spec.
- `provider-roc` and `omnia-client` have release cycles independent of product feature delivery. They are consumed as versioned artifacts, not actively co-developed with the services.
- The team needs to preserve independent versioning and deployment per component — a change to `drift-worker` should not force a release of `api-server`.
- The submodule detached HEAD problem causes engineers to unknowingly work on wrong commits.

---

## Considered Options

- **Option A** — Umbrella repo with git submodules (current state)
- **Option B** — True monorepo: all services as plain directories in one repo
- **Option C** — Polyrepo: each service fully independent, no umbrella coordinator

---

## Decision Outcome

We will adopt a **hybrid monorepo**: consolidate all components that change together as a
unit into `ucp-platform` as plain directories (no submodules), while keeping components
with independent lifecycles in their own repositories — referenced as versioned Go module
dependencies, not submodules.

### Component Disposition

**Consolidated into `ucp-platform` (plain directories, no submodules):**

| Component | Action | Notes |
|---|---|---|
| `api-server` | Collapse submodule → directory | Rewrite starts fresh |
| `temporal-worker` | Collapse submodule → directory | Rewrite starts fresh |
| `cli` | Collapse submodule → directory | Rewrite starts fresh |
| `crossplane/` | Already in-repo, no change | — |
| `k8s/` | Already in-repo | Contains only local dev setup scripts. Rename (`k8s-local-setup/` or similar) open for team discussion. |

**Remain as independent repositories (no submodule relationship):**

| Component | How referenced | Reason |
|---|---|---|
| `provider-roc` | Go module dep in `go.mod` + OCI image tag in Crossplane configs | Ships as a versioned Crossplane provider; independent release lifecycle |
| `omnia-client` | Go module dep in `go.mod` | Regenerated from external Omnia spec; version-pinned, not co-developed |
| `frontend` | Not referenced | Out of MVP scope; separate stack regardless |

**`docs` — open for team discussion:**

The docs repo is the team knowledge base. Removing it as a submodule eliminates
pointer-update friction. Two alternatives to maintain access for both engineers and tooling:

- **Option A — Skill-based fetch:** Remove submodule. A Claude Code skill clones the docs repo on demand locally. Re-run to stay in sync.
- **Option B — GHE MCP:** Remove submodule. Wire a GHE MCP connection to the docs repo so engineers and AI tooling can query it directly without a local clone. Token cost and query efficiency need evaluation before adopting.

### Justification

The three core services change together on every feature. Co-locating them eliminates
continuous coordination overhead without sacrificing deployment independence — independent
deployment is achieved via CI path filtering and tag-prefixed releases, not separate
repositories. `provider-roc` and `omnia-client` stay separate because their lifecycles
are genuinely independent: they are consumed as versioned dependencies, not evolved
alongside the services.

### Target Structure

```
ucp-platform/
├── go.work                       ← cross-module workspace (no replace directives)
├── api/
│   └── openapi.yaml              ← spec, source of truth for api-server + cli
├── api-server/                   ← Go module
│   ├── go.mod
│   ├── cmd/server/
│   └── internal/
├── temporal-worker/              ← Go module
│   ├── go.mod
│   ├── cmd/provisioning-worker/
│   ├── cmd/drift-worker/
│   ├── contracts/                ← public package, imported by api-server
│   └── internal/
├── cli/                          ← Go module
│   ├── go.mod
│   ├── cmd/
│   └── internal/
├── crossplane/
└── k8s/
```

### Migration

This decision coincides with a planned rewrite. History is not migrated — existing
submodule repositories are archived and the rewrite starts fresh.

```bash
git submodule deinit backend/api-server
git submodule deinit backend/temporal-worker
git submodule deinit cli
git rm backend/api-server backend/temporal-worker cli

mkdir -p api-server temporal-worker cli api
go work init ./api-server ./temporal-worker ./cli

git add .
git commit -m "chore: restructure as monorepo for api-server, temporal-worker, cli"
```

---

## Pros and Cons of the Options

### Option A — Umbrella Repo with Git Submodules

**Good:**
- Each submodule repo is self-contained with its own CI pipeline.
- Access control can be managed independently per repo.

**Bad:**
- Every cross-cutting feature requires 3–4 PRs across separate repos plus an umbrella pointer update. Services can be in an inconsistent state mid-delivery.
- Submodule detached HEADs cause engineers to unknowingly work on wrong commits.
- The OpenAPI spec lives in one submodule but is consumed by another — keeping them in sync requires manual coordination with no automated enforcement.
- `go.work` does not work cleanly with submodule detached HEADs, so cross-module local development requires `replace` directives instead.
- The umbrella repo itself does nothing except track SHA pointers — it pays the cost of both monorepo (one place to look) and polyrepo (separate CLs) without the benefit of either.

**CI/CD impact:** Each submodule has its own pipeline. A feature touching all three services opens 3 pipelines independently, with no atomic validation. Updating the umbrella pointer is a fourth commit with its own merge conflict surface.

### Option B — True Monorepo

**Good:**
- A cross-cutting feature is one PR, one CI run, one merge. Reviewers see the complete change atomically.
- The OpenAPI spec lives at a neutral root path (`api/openapi.yaml`). A spec change automatically triggers CI for both `api-server` and `cli` in the same run.
- The `temporal-worker/contracts/` package is imported by `api-server` as a typed Go dependency. Workflow name typos and input type mismatches are compile errors, not runtime failures.
- `go.work` resolves cross-module imports transparently in local development — no `replace` directives, no detached HEADs.
- One self-hosted CI runner serves the entire platform.
- Independent deployment per component is preserved via tag-prefixed releases (`api-server/v1.3.0`, `provisioning-worker/v1.1.0`, `drift-worker/v0.4.0`).

**Bad:**
- CI path filters in workflow files must be actively maintained. Adding a new worker binary requires a new workflow file and updating shared-path entries in existing ones.
- Go's `internal/` directory prevents accidental coupling between services, but the structural discipline must still be maintained — a monorepo makes it easier to reach across boundaries.

**CI/CD impact:** One repo, one runner, multiple workflow files with path filters. GitHub evaluates each workflow independently against the diff. Only affected pipelines run. Generated code is not committed — `make generate` runs as the first step of every CI job.

### Option C — Polyrepo

**Good:**
- Each service is fully self-contained with clean independence.
- No umbrella repo to maintain.

**Bad:**
- Same cross-cutting PR coordination problem as Option A, without even the single-clone convenience of the umbrella.
- The OpenAPI spec has no natural home — it either lives in `api-server` (creating a directional dependency for `cli`) or in a fourth `ucp-api-contracts` repo (another repo to manage).
- No atomic validation of cross-cutting changes.

**CI/CD impact:** Same as Option A without the umbrella noise. Still requires multiple PRs and multiple pipelines for cross-service features.

---

## Consequences

**Positive:**
- Cross-cutting feature delivery requires one PR instead of 3–4. Review is atomic and complete.
- The OpenAPI spec, workflow contracts, and service implementations are always validated together by CI.
- Local development across modules works without manual `replace` directive management.
- `provider-roc` and `omnia-client` are versioned dependencies — their releases are decoupled from service feature delivery.
- Each component retains independent versioning and deployment. A hotfix to `drift-worker` does not trigger a release of `api-server`.

**Negative:**
- CI workflow files require ongoing maintenance as new binaries are added.
- The coupling risk between co-located services increases slightly — Go's `internal/` package restriction mitigates this but does not eliminate the need for discipline.
- Migration requires archiving existing submodule repositories. Git history is not preserved across the rewrite.

---

## Open Questions

1. **`k8s/` folder rename** — the folder contains only local dev setup scripts, making the name ambiguous. Proposed alternatives: `k8s-local-setup/`, `local/`, or retain `k8s/`. Open for team discussion.

2. **`docs` submodule** — remove in favour of skill-based fetch (Option A) or GHE MCP (Option B)? Both need evaluation. The chosen approach should not degrade access to the knowledge base for engineers or tooling.
