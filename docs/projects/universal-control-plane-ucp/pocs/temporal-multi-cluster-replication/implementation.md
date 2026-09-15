---
title: "Temporal Multi-Cluster Replication — Implementation"
space: UCP
parent_page_id: "../temporal-multi-cluster-replication.md"
---

# Temporal Multi-Cluster Replication — Implementation

Supporting proof doc. Describes what was built, run, and observed for each phase in
[design.md](design.md). See [poc-report.md](poc-report.md) for the verdict.

## Source Code

- **Repository:** `aripermana-putra/kitchen-sink`
- **Path:** `temporal-multi-cluster-replication-poc/`
- **Files:** `cluster-a/docker-compose.yml`, `cluster-a/config_template.yaml`,
  `cluster-b/docker-compose.yml`, `cluster-b/config_template.yaml`, `README.md`
- **Commits:** `41aa54c` (docker-compose/config + runnable playbook), `eb63069` (Phase 16–17
  worker: `worker/main.go`, `worker/go.mod`, `worker/go.sum`), `cf2c5d0` (the retry-on-connect
  fix to `worker/main.go` that made the second pass of Phase 17 reproduce the dispute)

## Environment

| | Cluster A | Cluster B |
|---|---|---|
| Docker context | `ucp-crossplane` Colima VM | `temporal-mcr` Colima VM (new, dedicated) |
| Frontend, reachable from the host | `127.0.0.1:7233` | `127.0.0.1:8233` |
| Frontend, reachable from the other cluster's container | `host.docker.internal:7233` | `192.168.5.2:7233` (Colima `vz` driver's guest default gateway) |
| Cluster identity | `cluster-a`, `initialFailoverVersion: 1` | `cluster-b`, `initialFailoverVersion: 2` |
| `numHistoryShards` | 4 | 4 |
| Image | `temporalio/auto-setup:1.25.2` | `temporalio/auto-setup:1.25.2` |
| Persistence | Postgres 13, no Elasticsearch | Postgres 13, no Elasticsearch |

Both clusters share `failoverVersionIncrement: 100`. The `Cluster A → Cluster B` and
`Cluster B → Cluster A` addresses in the table above are deliberately different strings —
each is the address that side can actually reach, not a shared value (see Phase 5 below and
[design.md](design.md)'s note on this).

## Phase-by-phase record

### Phase 1–2 — Stand up both clusters

`docker compose up -d` attaches to whatever Docker context is active at invocation time
rather than the bare host by default. In this run, that context was the `ucp-crossplane`
Colima VM for Cluster A; Cluster B was brought up explicitly inside the new, dedicated
`temporal-mcr` Colima profile. Both stacks came up on the first attempt with the config
below.

One config fix was required on both clusters before the server would start: the initial
`config_template.yaml` used a simplified `archival: {state: disabled}` shape, which failed
Temporal's config validator (`config validation error: invalid history archival config`).
Temporal requires the full `archival.provider` block to be present even when archival is
disabled — fixed by including the `filestore` provider block (as shown in the committed
`config_template.yaml` files) on both clusters.

### Phase 3 — Reachability

Confirmed from inside each cluster's actual container process, not just a host shell:

- `nc -zv 192.168.5.2 7233` from inside Cluster B's container → `open`, and a `curl` against
  it returned a real application-layer response from Cluster A's frontend.
- Cluster A reachable from the host at `127.0.0.1:7233` via Colima's normal port-forwarding.

No vmnet bridging, no relay, no changes to `bindOnIP` (both clusters use the default
`"0.0.0.0"`) were needed for either direction to work.

### Phase 4 — Cluster metadata

Confirmed present in both `config_template.yaml` files as committed: `enableGlobalNamespace:
true`, `failoverVersionIncrement: 100`, distinct `initialFailoverVersion` (1 / 2),
`numHistoryShards: 4` on both.

### Phase 5 — Cluster registration

```shell
temporal --address 127.0.0.1:7233 operator cluster upsert --frontend-address "host.docker.internal:8233" --enable-connection --enable-replication
temporal --address 127.0.0.1:8233 operator cluster upsert --frontend-address "192.168.5.2:7233" --enable-connection --enable-replication
```

The first attempt used `127.0.0.1:8233` for the A→B direction and failed with `connection
refused` — the dial happens from *inside Cluster A's own container*, where `127.0.0.1` is
that container's own loopback, not the host's. Fixed with `host.docker.internal:8233`.

`temporal operator cluster list` on both sides subsequently shows both `cluster-a` and
`cluster-b`. `IsReplicationEnabled` displays as `false` in that listing on both sides even
after `--enable-replication` — this did not affect Phases 6–7 below, which proved replication
was functioning regardless; treated as a display-only quirk of this CLI/server version, not
a functional block.

### Phase 6 — Global namespace

Namespace `mcr-poc` created global, active on `cluster-a`. Visible from `cluster-b` within 5
seconds via `temporal --address 127.0.0.1:8233 operator namespace describe mcr-poc`, with a
matching `FailoverVersion: 1`.

### Phase 7 — Workflow replication

`mcr-poc-wf-1` started against `cluster-a` (no worker running — only the
`WorkflowExecutionStarted`/`WorkflowTaskScheduled` events are needed to prove replication).
Visible via `temporal --address 127.0.0.1:8233 workflow show --namespace mcr-poc
--workflow-id mcr-poc-wf-1` within the first 3-second poll.

### Phase 8 — Manual flip while Cluster A is healthy

`temporal --address 127.0.0.1:7233 operator namespace update --namespace mcr-poc
--active-cluster cluster-b`, run against `cluster-a` while it was still running. `cluster-a`'s
own `namespace describe` showed `cluster-b` active with **zero observable lag** on repeated,
immediate polling. This is the direct mechanical consequence of the flip writing to whichever
cluster receives the command — the originating side never has a stale-belief window; only a
non-originating side learning about the change via replication would. `FailoverVersion`
became `101` (the smallest value ≡ 1 mod 100 that is ≥ 2, per the versioning rule — since
`cluster-b`'s own lane is ≡ 2 mod 100, not ≡ 1; the actual failover version tracks whichever
cluster is newly active, using *its own* lane).

New workflow start against `cluster-b` in `mcr-poc` succeeded immediately after.

### Phase 9 — Forced failover with Cluster A stopped

Namespace flipped back to `cluster-a` active first, confirmed. Cluster A's containers then
stopped entirely; `connection refused` against `127.0.0.1:7233` confirmed it was genuinely
unreachable, not just flipped. `temporal --address 127.0.0.1:8233 operator namespace update
--namespace mcr-poc --active-cluster cluster-b` run against `cluster-b` (the only reachable
cluster). `FailoverVersion` became `102`. A new workflow (`mcr-poc-wf-3`) started against
`cluster-b` standalone, with `cluster-a` fully down.

### Phase 10 — Rejoin

Cluster A's containers restarted, no manual step beyond starting the process. `temporal
--address 127.0.0.1:7233 operator namespace describe mcr-poc` already showed `cluster-b`
active with matching `FailoverVersion: 102` at the very first check, 10 seconds after
restart — bounded by container boot time, not by any additional replication-catch-up delay.
`mcr-poc-wf-3` (started on `cluster-b` while `cluster-a` was down) was also fully visible from
`cluster-a` after rejoin, confirming the backlog replicated in, not just the namespace flag.

### Phase 11 — Namespace handover

Namespace flipped back to `cluster-a` active first (`FailoverVersion: 201`). The internal
`namespace-handover` workflow, not documented as an operator-facing CLI verb, was started
directly as a plain workflow execution:

```shell
temporal --address 127.0.0.1:7233 workflow start \
  --namespace temporal-system \
  --task-queue default-worker-tq \
  --type namespace-handover \
  --workflow-id mcr-poc-handover-1 \
  --input '{"Namespace":"mcr-poc","RemoteCluster":"cluster-b","AllowedLaggingSeconds":10,"AllowedLaggingTasks":0,"HandoverTimeoutSeconds":30}'
```

`namespace-handover` (v1) and `namespace-handover-v2` are two registered workflow types for
the same underlying logic, not two different mechanisms — confirmed from source
(`handover_workflow.go`): v2 is a thin wrapper that adds one guard (rejects the run if a
workflow run timeout was set) and then calls straight into v1's implementation. The plan was
to try v1 first and fall back to v2 only if v1 was rejected for a version-mismatch reason;
v1 was accepted outright, so v2 was never needed.

Input fields (`NamespaceHandoverParams`, from source):

| Field | Meaning |
|---|---|
| `Namespace` | The namespace to hand over — `mcr-poc` here |
| `RemoteCluster` | The cluster to hand over *to* — `cluster-b`, the target that becomes active |
| `AllowedLaggingSeconds` | How far behind `RemoteCluster` is allowed to be, in replication lag, before the workflow proceeds into the `HANDOVER` state at all |
| `AllowedLaggingTasks` | Same idea as above, but measured in outstanding replication tasks rather than seconds |
| `HandoverTimeoutSeconds` | How long to wait for `RemoteCluster` to fully drain to zero lag once inside `HANDOVER` state, before giving up and resetting back to normal without completing the flip |

Accepted with no caller/permission rejection. The workflow's own history matches
`handover_workflow.go` step-for-step:

| Step | Activity | Duration |
|---|---|---|
| 1 | `GetMetadata` | ~2ms |
| 2 | `GetMaxReplicationTaskIDs` | ~2ms |
| 3 | `WaitReplication` | ~2ms (already caught up — no lag to wait on) |
| 4 | `UpdateNamespaceState` → `HANDOVER` | ~8ms |
| 5 | `WaitHandover` | ~2.0s (only real wait in the run) |
| 6 | `UpdateActiveCluster` | ~9ms |
| 7 | `UpdateNamespaceState` → reset to normal | ~7ms |

Total wall-clock ~2.1 seconds; workflow status `COMPLETED`. Final state on both clusters:
`ActiveClusterName: cluster-b`, `FailoverVersion: 202`, `ReplicationConfig.State:
Unspecified` (correctly reset, not left in `HANDOVER`).

Live client-visible rejection of a request during the `HANDOVER` window was not
independently observed — with zero pre-existing replication lag, the window between step 4
and step 7 lasted only ~2 seconds total, shorter than the 1-second polling granularity used
to watch for it. The window's existence is still proven by the `UpdateNamespaceState`
call pair in the workflow history.

### Phase 12 — Attempting to force real replication lag

Follow-up specifically to close the gap above, by forcing a real, sustained backlog rather
than relying on 1-second polling to get lucky. Tested systematically, not assumed symmetric:

| Attempt | Flags set to `false` | Where | Result |
|---|---|---|---|
| 1 | `--enable-connection` | Cluster B's registration of Cluster A | A workflow started on A still replicated to B within the usual few seconds |
| 2 | `--enable-connection` | Both clusters' registrations of each other (waited 15s to rule out delay) | Still replicated |
| 3 | `--enable-connection` and `--enable-replication` | Both clusters, both flags | Still replicated |

`operator cluster list` confirmed the flags genuinely registered as `false` throughout each
attempt — not a case of the command silently failing to apply. **The already-established
replication stream is not gated by these metadata toggles.** No further escalation attempted
(e.g. firewall-level port blocking, killing the stream process) — those would test something
other than the documented CLI mechanism itself. Flags restored to `true`/`true` on both
clusters afterward; confirmed normal operation resumed.

Steps 2–5 of the planned Phase 12 (spam a backlog while disabled, re-enable, observe
rejection) were not reached, since there is no backlog to build if disabling does not
actually stop replication.

### Phase 13 — Single-cluster with pre-existing history

Reused the existing, already-running cluster-a (no fresh instances needed — a running
cluster's `enableGlobalNamespace: true` is a capability flag, it doesn't force every
namespace on it to be global). Created a local namespace, `mcr-poc-migration`, and started 5
workflows against it: `migration-pre-1` through `migration-pre-5`.

### Phase 14 — Introducing Cluster B and promoting the namespace

Two CLI surface corrections from what source suggested:

- The field is `UpdateNamespaceRequest.PromoteNamespace` in source, but the `temporal` CLI's
  actual flag is **`--promote-global`**.
- `--promote-global` and `--cluster` **cannot be combined in one call** — the CLI itself
  states the other flag will be omitted. Required two separate calls:
  ```shell
  temporal --address 127.0.0.1:7233 operator namespace update --namespace mcr-poc-migration --promote-global
  temporal --address 127.0.0.1:7233 operator namespace update --namespace mcr-poc-migration --cluster cluster-a --cluster cluster-b
  ```

A first attempt to confirm the pre-existing workflows were absent from cluster-b — via
`temporal --address 127.0.0.1:8233 workflow list`/`workflow show` — appeared to show them
present immediately. This was a false positive: both clusters' configs carry
`dcRedirectionPolicy: policy: "all-apis-forwarding"` (copied from Temporal's own reference
config in Phase 1), which forwards every frontend call — including visibility/list queries —
from a standby cluster straight to the active one. Querying cluster-b was silently just
relaying cluster-a's answer. Confirmed via direct Postgres inspection instead:

```sql
SELECT workflow_id, run_id FROM current_executions WHERE namespace_id = '<namespace-id>';
```

run against cluster-b's own database — showed only `migration-post-1` (started after
promotion, to confirm normal replication still worked) before backfill; none of the
`migration-pre-*` rows.

One additional consequence of `all-apis-forwarding` discovered in the process: with
cluster-a stopped, the same forwarded calls against cluster-b don't fall back to local
data — they hang until gRPC deadline (`context deadline exceeded`, ~10s) instead. A standby
under this policy cannot serve any frontend request once its peer is unreachable, visibility
included.

### Phase 15 — Backfilling with `force-replication`

```shell
temporal --address 127.0.0.1:7233 workflow start \
  --namespace temporal-system \
  --task-queue default-worker-tq \
  --type force-replication \
  --workflow-id mcr-poc-force-replication-1 \
  --input '{"Namespace":"mcr-poc-migration","Query":"WorkflowId != \"\"","TargetClusterName":"cluster-b"}'
```

`Query` uses the same SQL-like list-filter syntax as `workflow list --query`, confirmed
against that command first rather than assumed. `force-replication` (v1) accepted and
completed on the first attempt — no need for v2. `TargetClusterName` alone was sufficient;
`EnableVerification`/`TargetClusterEndpoint` were not needed. Completed in ~11 seconds
(07:56:49–07:57:00) for the 5 backfilled workflows. The same direct Postgres query against
cluster-b afterward showed all 6 workflows present, correct `run_id`s.

### Phase 16 — Real worker infrastructure

`worker/main.go` (kitchen-sink): one workflow `DisputeWorkflow` executing one activity
`SlowActivity` (`StartToCloseTimeout` 20s, sleeps 15s, logs `CLUSTER_LABEL` and attempt
number). `go mod tidy` resolved `go.temporal.io/sdk v1.49.0`. `go build` produced a 28.2MB
binary — larger than the design doc's rough estimate; added to the kitchen-sink root
`.gitignore` (`temporal-multi-cluster-replication-poc/worker/dispute-worker`), confirmed
excluded via `git check-ignore -v`.

### Phase 17 — The dispute attempt

**First attempt — a timing artifact, not a real result.** Confirming the activity was
in-flight and killing the worker/cluster were split across separate tool calls; the
inter-call latency alone (~90 seconds) let the activity's 15-second sleep complete and report
back to Cluster A *before* the kill command ran — event timestamps proved it
(`ActivityTaskCompleted` at 00:05:04Z, containers stopped at 00:06:35Z). No outage actually
occurred. Fixed by chaining start→confirm→kill→stop within one shell invocation for the
second attempt.

**Second attempt — genuine outage, clean reproduction of the retry/completion half.**
`worker-a` `SIGKILL`'d and cluster-a's containers stopped ~4 seconds into the activity's
15-second sleep — worker log confirmed no completion line before the kill. Forced failover
to cluster-b. `worker-b` polled and received the retry immediately: log shows `attempt=2
cluster=B executing`, completed 15s later. `workflow show`: `Status: COMPLETED`, `Result:
"completed by cluster=B attempt=2"`.

**The dispute half did not reproduce on this first pass, for a worker-code reason, not a
Temporal-mechanism one.** `newClient()` called `client.Dial(...)` followed by `log.Fatalln`
on any error, with no retry. When `worker-a` was launched racing `docker compose start
cluster-a`, cluster-a's frontend wasn't yet accepting connections — `Dial` failed with
connection-refused, and the process exited immediately, before ever reaching its poll loop.
It never attempted the race. A second variation (wait for cluster-a's `operator cluster
health` to report `SERVING`, then launch `worker-a`) missed the window from the other side —
by the time health-polling confirmed `SERVING`, cluster-a's own namespace belief had already
converged to `cluster-b`, consistent with Phase 10's ~10-second convergence. `workflow show`
against both clusters for `dispute-wf-3` returned identical 11-event history and result — no
divergence, because no dispute occurred on this pass.

### Phase 17, second pass — the dispute, actually forced and observed

Fixed `newClient()` (`worker/main.go`) to retry `client.Dial` for up to 2 minutes (1s between
attempts, logging each failure) instead of failing fast — committed as `cf2c5d0`, rebuilt,
confirmed compiling before re-running.

Fresh workflow (`dispute-wf-4`), same mechanism as the first pass: activity sleeping from
02:16:21; `worker-a` `SIGKILL`'d and cluster-a's containers stopped at 02:16:26 (~5s in).
Forced failover to cluster-b; `worker-b` retried immediately and completed:
`"completed by cluster=B attempt=2"`, `FailoverVersion: 202`.

`worker-a` relaunched racing `docker compose start cluster-a`. Log evidence, in order:
`client.Dial to 127.0.0.1:7233 failed (connection refused), retrying in 1s...` → connected on
the next attempt → **immediately handed the activity again**: `attempt=2 cluster=A executing
(sleeping 15s)`. Ran the full duration, logged `completed by cluster=A attempt=2` locally,
then on reporting the result back to cluster-a received:
```
Task processing failed with error ... Error workflow execution already completed
```
A clean, explicit server-side rejection — not a silent overwrite, not a hang.

Querying cluster-a's own history directly afterward: 11 events, byte-for-byte identical to
cluster-b's, carrying cluster-b's timestamps (`ActivityTaskStarted` 02:16:43,
`ActivityTaskCompleted` 02:16:58) rather than cluster-a's local re-execution window
(02:29:02–02:29:17). No trace of cluster-a's second attempt survived. Final state on both
clusters: `Status: COMPLETED`, `Result: "completed by cluster=B attempt=2"`, identical
history — this time because the dispute actually occurred and was resolved, not because
nothing happened.

Not investigated: whether the discarded branch is visible transiently anywhere (a DLQ, a
replication-task table) before cleanup, versus never persisted at all. The observation here
is "no trace found when queried after the fact," not "watched the discard happen in real
time."

## Notes for reproducing this PoC

- `namespace-handover` is not exposed as a documented CLI verb for self-hosted operators — it
  was invoked as a plain `workflow start` against Temporal's own `temporal-system` namespace,
  reverse-engineered from source. A production runbook would need either this same approach
  documented internally, or confirmation from Temporal on the intended operator-facing path.
- Don't trust a query against a standby cluster to reflect what that standby actually has
  locally — with `dcRedirectionPolicy: all-apis-forwarding` (used throughout this PoC), it's
  forwarded to the active cluster, not served from local data. Verifying anything about a
  standby's own state specifically needs either a different redirection policy, or querying
  its persistence store directly, as done in Phase 14.
- `--promote-global`, not `--promote-namespace`, and it cannot be combined with `--cluster` in
  the same `operator namespace update` call — two separate calls are required.
- For any timing-sensitive test (confirm-in-flight → kill → failover, or similar), chain the
  steps within a single shell invocation rather than separate tool/command calls — inter-call
  latency alone can be enough to let the thing you're trying to interrupt finish first (see
  Phase 17's first attempt).
- A worker with no retry-on-connect logic cannot participate in any test that races its
  startup against a cluster becoming reachable — a plain `client.Dial()` + fatal-on-error
  worker will simply die rather than wait and retry. `worker/main.go`'s `newClient()` now
  retries for up to 2 minutes (1s between attempts) — confirmed this is what let the actual
  dispute reproduce on the second pass.
