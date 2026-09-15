---
title: "Temporal Multi-Cluster Replication — PoC Report"
space: UCP
parent_page_id: "../temporal-multi-cluster-replication.md"
---

# Temporal Multi-Cluster Replication — PoC Report

Human-first verdict document.

| | |
|---|---|
| **Answers research question** | [research/temporal-multi-cluster-replication.md](../../research/temporal-multi-cluster-replication.md) — specifically, whether self-hosted Temporal's Multi-Cluster Replication (MCR) actually works: cluster-to-cluster connectivity, namespace/history replication, and failover, including recovery from a real outage |
| **Design** | [design.md](design.md) |
| **Implementation detail** | [implementation.md](implementation.md) |
| **Verdict** | MCR works as documented, including the specific behavior the research doc's Recommendation depends on |

## What this PoC proved

Two independent, independently-persisted Temporal clusters establish a working MCR
relationship using only the connectivity described in
[design.md](design.md) — no vmnet bridging, no relay, no special network configuration
beyond registering each cluster's frontend address at the other. Every phase in the design
passed:

- Bidirectional cluster registration (`cluster upsert --enable-connection
  --enable-replication`).
- A global namespace created on one cluster, visible on the other within seconds, with a
  matching failover version.
- A workflow started on the active cluster, with its history visible on the passive cluster
  shortly after — with no worker running anywhere, confirming this is replication, not
  workflow execution.
- **A manual flip while both clusters are healthy**, with zero observable lag on the
  cluster that issued the command — the flip writes directly to whichever cluster receives
  it; only a non-originating cluster learning about the change via replication would see any
  delay.
- **A forced failover with the active cluster actually stopped** — the passive cluster
  accepted the flip and a new, standalone workflow start on its own, with the other cluster
  genuinely unreachable, not just flipped.
- **Rejoin after the forced failover, with no manual intervention** — restarting the failed
  cluster's containers was the only action taken. Within the first check (10 seconds after
  restart, bounded by container boot time), the rejoined cluster's own view already matched
  the other cluster's active-cluster state and failover version, and a workflow started on
  the other cluster while it was down was fully visible after rejoin.

That last point is the one that matters most. The research doc's Recommendation rests
specifically on MCR self-healing on rejoin without a manual rebuild, in contrast to
Cloud SQL cross-region replication's delete-and-recreate recovery path. This PoC exercised
exactly that scenario, deliberately, and it held.

- **The internal `HANDOVER`-state drain workflow (Phase 11)** — a different, more specific
  mechanism than the plain metadata flip used in Phases 8–9 — ran successfully end to end as
  a plain `workflow start` against Temporal's own `temporal-system` namespace: metadata
  check, replication-watermark check, drain wait, `HANDOVER` state entered, active cluster
  flipped, state reset to normal. Total wall-clock ~2.1 seconds, with a correct final
  `FailoverVersion` and no residual `HANDOVER` state stuck on either side. It was not
  documented anywhere as an operator-facing CLI verb — this was invoked directly as an
  internal workflow type, reverse-engineered from source, and it worked without any
  caller/permission rejection.
- **Adding a cluster to an already-populated deployment (Phases 13–15)** — a scenario UCP
  will not actually face (both clusters are planned from the start), but tested for team
  reference. Pre-existing workflows, created before Cluster B was ever registered, were
  confirmed genuinely absent from Cluster B's own database after promotion — and confirmed
  present, correctly, after running the separate `force-replication` migration workflow
  (~11 seconds for 5 workflows). Both the absence and the presence were confirmed by querying
  Cluster B's Postgres directly, not through the Temporal API — see the finding below on why
  that mattered.
- **A real worker resuming stale work on the passive cluster (Phases 16–17, half of it).**
  Every earlier phase only checked that data replicated, never that a workflow was actually
  executed. This is the first real, worker-driven confirmation: an activity genuinely
  in-flight on Cluster A (worker mid-sleep) when Cluster A was killed and failed over to
  Cluster B — the timed-out attempt was retried and completed by a real worker on Cluster B,
  with the workflow reaching `COMPLETED` carrying that worker's result. Not just inferred from
  source or CLI history inspection this time — an actual second attempt, actually executed, by
  an actual worker process on the other cluster.

## What this PoC did not prove

- **A genuine version-conflict dispute (the other half of Phases 16–17) did not occur —
  and not for a Temporal-mechanism reason.** The plan was to have a worker on Cluster A,
  already running and polling the instant Cluster A's frontend became reachable again after
  restart, race Cluster A's own stale-belief window and get handed the same activity a second
  time, producing a genuine duplicate execution. It didn't happen because the test's own
  worker has no retry-on-connect logic: `client.Dial()` fails fast and fatally on connection
  refused, so when launched right as Cluster A's container was restarting, the worker process
  died before it ever reached its poll loop — it never got a chance to attempt the race at
  all. A second attempt (wait for `SERVING` first, then launch) missed the window from the
  other side — by the time health-polling confirmed `SERVING`, Cluster A's namespace belief
  had already converged to `cluster-b`, consistent with Phase 10's ~10-second convergence
  finding. Both clusters agreed exactly on the workflow's final state throughout, because no
  divergence ever actually occurred. A follow-up that wants to force this for real needs a
  worker built to survive a briefly-unreachable cluster (retry-on-connect, or
  `client.NewLazyClient`'s deferred-connection behavior) rather than one that dies on first
  contact failure — this is a gap in the test harness, not evidence about MCR's own
  conflict-resolution mechanism one way or the other.
- **Live client-visible rejection during the `HANDOVER` window was not independently
  observed.** The workflow's own history proves the window existed (the
  `UpdateNamespaceState(HANDOVER)` → `UpdateNamespaceState(NORMAL)` pair is unambiguous), but
  because both clusters already had zero replication lag, the window lasted only ~2 seconds
  total, too short for 1-second-granularity polling to catch a request landing inside it.
- **Forcing real replication lag via `cluster upsert --enable-connection=false` /
  `--enable-replication=false` does not work.** This was tested directly, as a follow-up
  specifically to close the gap above: setting either or both flags, on either or both
  clusters, confirmed registered as `false` via `cluster list`, did not stop an
  already-established replication stream — a workflow started on Cluster A while both flags
  were `false` on both sides still replicated to Cluster B within the usual few seconds, even
  after a 15-second wait to rule out delayed effect. This lines up with the earlier finding
  that the replication stream's existence is decoupled from cluster metadata — that
  decoupling apparently extends to these specific toggles too, not just to active/passive
  status. A real test of the live-rejection behavior would need actual network-level
  partitioning between the clusters, not a metadata flag — this PoC did not attempt that.
- **Cloud SQL was not tested.** Both clusters ran on local Postgres containers. Nothing here
  validates Cloud SQL-specific behavior (regional HA interaction, IAM/network policy,
  Cloud SQL Proxy, connection limits) — only Temporal's own cluster-to-cluster mechanics,
  which are independent of the underlying database engine.
- **No load or throughput testing.** One namespace, a handful of workflows, no concurrent
  traffic. Replication lag under real load, and its effect on RPO, is not characterized by
  this PoC.
- **No mTLS/auth between clusters.** Plaintext gRPC only, as scoped.

## Four operational findings worth carrying forward

- **Querying a standby cluster does not necessarily tell you what the standby actually has.**
  With `dcRedirectionPolicy: all-apis-forwarding` (the policy this PoC used throughout, copied
  from Temporal's own reference config), a standby forwards *every* frontend call — including
  visibility/list queries — to the active cluster. The first attempt to confirm Phase 14's
  pre-existing workflows were absent from Cluster B *appeared* to show them present,
  immediately, because the query against B was silently just relaying A's answer. The only way
  to get a genuinely local answer was to query Cluster B's own Postgres `current_executions`
  table directly, bypassing the Temporal API entirely. This also means a standby under this
  policy cannot serve *any* frontend request once its peer is unreachable — the forwarded call
  just hangs until timeout rather than falling back to local data. This corrects a claim in
  the research doc ("visibility APIs work against both active and standby clusters") that was
  stated as a general MCR property but is actually conditional on `dcRedirectionPolicy` — now
  qualified there.
- **`cluster upsert --enable-connection=false` / `--enable-replication=false` do not sever
  an already-established replication stream.** Whatever these flags gate, it isn't the live
  stream itself once it's running — confirmed by direct testing (see above), not just
  inferred. Anyone relying on this flag to actually pause replication (e.g. for a maintenance
  window) should verify that assumption first; it did not hold here.
- **A container's own `127.0.0.1` is not the host's `127.0.0.1`.** Registering a peer
  cluster's address from *inside* a Temporal server's own container needs
  `host.docker.internal` (or the equivalent reachable address), not the loopback address a
  human would use to test reachability from a shell. This generalizes past Colima: the
  address a `cluster upsert` command needs is the address the *server process* can reach, not
  necessarily the address a human tested with.
- **`docker compose up -d` attaches to whatever Docker context is currently active, not
  necessarily the one you assume.** Worth an explicit `docker context ls` check before
  bringing up a stack when multiple contexts exist.

## Recommendation

No change to the research doc's Recommendation — this PoC was validation for it, and the
validation held on both claims that mattered most (self-healing rejoin, and a working
graceful planned-failover path). Before this is adopted as UCP's production direction, the
still-open items are: testing against Cloud SQL specifically, confirming the live-rejection
behavior during a real `HANDOVER` window using actual network-level partitioning to force
lag (the metadata-flag approach tried here does not work), finding or documenting the correct
operator-facing way to trigger handover in a production runbook (this PoC invoked an
undocumented internal workflow type directly), deciding the actual `dcRedirectionPolicy` value
for UCP's production deployment with the forwarding-vs-local-serving trade-off now understood
(not previously an identified decision point), and resolving MCR's
experimental-support-status risk (unchanged from the research doc, not something this PoC
could resolve either way). Since UCP's actual rollout creates both clusters from the start,
the Phases 13–15 finding (pre-existing data isn't backfilled automatically) does not block
anything — it's reference material for a scenario that isn't the planned path.

One item moves from "confirmed" to "still open, with a known cause" after Phases 16–17: the
version-based conflict-resolution mechanism (highest failover version wins, losing branch
discarded) that the research doc describes from source has still never actually been forced
to occur and observed — only reasoned about. This PoC's attempt to force it failed for a
test-harness reason (the worker's lack of retry-on-connect), not because MCR behaved
differently than documented. It's a genuine remaining gap before treating the
conflict-resolution mechanism as proven rather than well-sourced-but-theoretical — worth a
follow-up with a more resilient worker if that distinction matters for the eventual
production decision.

## Test Data

Cluster identities (`temporal operator cluster list`):

| Cluster | `HistoryShardCount` | `InitialFailoverVersion` |
|---|---|---|
| cluster-a | 4 | 1 |
| cluster-b | 4 | 2 |

Failover version progression observed across the PoC (`temporal operator namespace describe
mcr-poc`), consistent with the versioning rule in the research doc:

| Event | Active cluster | `FailoverVersion` |
|---|---|---|
| Namespace created | cluster-a | 1 |
| Manual flip (Phase 8, A healthy) | cluster-b | 101 |
| Flipped back to A, then forced failover (Phase 9, A stopped) | cluster-b | 102 |
| Rejoin (Phase 10) — cluster-a's own view after restart | cluster-b | 102 (matched, no divergence) |
| Flipped back to A for Phase 11, then handover to B | cluster-b | 202 |

Workflow IDs used: `mcr-poc-wf-1` (Phase 7, replication check), `mcr-poc-wf-3` (Phase 9,
started on cluster-b while cluster-a was down; confirmed visible from cluster-a after rejoin
in Phase 10).

Phase 11 (`namespace-handover`, workflow ID `mcr-poc-handover-1`) execution timeline, from
the workflow's own history:

| Step | Activity | Duration |
|---|---|---|
| 1 | `GetMetadata` | ~2ms |
| 2 | `GetMaxReplicationTaskIDs` | ~2ms |
| 3 | `WaitReplication` | ~2ms (already caught up — no lag to wait on) |
| 4 | `UpdateNamespaceState` → `HANDOVER` | ~8ms |
| 5 | `WaitHandover` | ~2.0s (the only real wait in the run) |
| 6 | `UpdateActiveCluster` | ~9ms |
| 7 | `UpdateNamespaceState` → reset to normal | ~7ms |

Total wall-clock: ~2.1 seconds. Neither `AllowedLaggingSeconds` (10) nor
`HandoverTimeoutSeconds` (30) directly explains the ~2s `WaitHandover` duration observed with
zero pre-existing lag — likely a fixed internal poll/heartbeat cadence in that activity, not
confirmed further.

Phases 13–15 (brownfield cluster addition): namespace `mcr-poc-migration`, created local on
cluster-a, 5 pre-existing workflows (`migration-pre-1` through `migration-pre-5`). After
promotion to global and cluster registration, one new workflow (`migration-post-1`) confirmed
replicating normally. Direct Postgres query against cluster-b's `current_executions` table
confirmed only `migration-post-1` present before backfill; all 6 workflows present after
`force-replication` (workflow ID `mcr-poc-force-replication-1`) completed, ~11 seconds
wall-clock (started 07:56:49, completed 07:57:00) for the 5 backfilled workflows.

Phases 16–17 (real worker, dispute attempt): namespace `mcr-poc-dispute`. `dispute-wf-3`
started against cluster-a with `worker-a` (`CLUSTER_LABEL=A`) polling; `worker-a` killed
(`SIGKILL`) and cluster-a stopped ~4 seconds into the activity's 15-second sleep, confirmed
via worker log with no completion line before the kill. Forced failover to cluster-b;
`worker-b` (`CLUSTER_LABEL=B`) picked up the retry immediately (`attempt=2`), completed 15s
later. `workflow show` on cluster-a: `Status: COMPLETED`, `Result: "completed by cluster=B
attempt=2"`, 11-event history — identical on both clusters after cluster-a's restart, no
divergence. `worker-a`, restarted racing cluster-a's container boot, never re-entered the
poll loop — `client.Dial()` failed with connection-refused and the process exited via
`log.Fatalln` before attempting anything.
