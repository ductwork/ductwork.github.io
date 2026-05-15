---
title: Durability
parent: Advanced
nav_order: 15
---

# Durability

Ductwork is a *durable* pipeline framework: every unit of work and every state
transition is persisted to your relational database before it is acted upon.
There is no in-memory queue, no separate broker, and no work that exists only
inside a running process. If a worker, an advancer, the supervisor, or the
entire host disappears, the database still holds a complete and consistent
picture of what was in flight, and Ductwork is designed to recover from that
picture automatically.

This page explains the mechanisms that make that true: atomic claiming,
fencing (claim) tokens, heartbeats and the reaper, the database clock, the
two-phase advancement commit, and the manual `revive!` escape hatch.

{: .note }
Durability concepts span several database tables. If a term is unfamiliar,
see the [Lexicon](lexicon.html) and the data model diagram first.

## The durability model in one paragraph

Work is represented by rows, not messages. A triggered pipeline writes
`run`, `branch`, `step`, `job`, `execution`, and `availability` rows. Workers
and advancers do nothing more than *atomically claim* one of those rows, do the
work, and *atomically commit* the result, with each claim and commit guarded so that
exactly one process can win, and so that a process which fell behind can never
overwrite the winner. Any row left claimed by a process that stopped
heartbeating is swept up by the **reaper** and re-made available. The net
effect is **at-least-once execution** of every step with no lost work.

## Atomic claiming

The central invariant is that a given piece of work is owned by exactly one
process at a time. Ductwork enforces this with database-level atomic claims,
never with application-side coordination or locks held across the work itself.

### Claiming jobs (job workers)

When a job worker thread looks for work it claims an `availability` row, the
"this execution is ready to run" marker. The claim strategy adapts to the
database:

- **Row-locking (`RowLockingExecutionClaim`)**: used on PostgreSQL, MySQL /
  Trilogy, CockroachDB, and Oracle. The candidate row is selected with
  `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1`. `SKIP LOCKED` means concurrent
  workers transparently step over each other's locked rows instead of
  contending or blocking, so throughput scales with worker count.
- **Optimistic locking (`OptimisticLockingExecutionClaim`)**: the fallback for
  databases without `SKIP LOCKED` (e.g. SQLite). A candidate id is read, then
  claimed with a conditional write:

  ```sql
  UPDATE ductwork_availabilities
     SET completed_at = ?, process_id = ?
   WHERE id = ? AND completed_at IS NULL
  ```

  The claim succeeds only if exactly one row was updated. If another worker won
  the race, zero rows update and the worker simply loops: no error, no double
  execution.

In both strategies the winning worker stamps its `process_id` onto both the
`availability` and the `execution` row. That `process_id` is the basis for the
commit guard described [below](#the-execution-commit-guard).

{: .note }
Candidate selection uses the [database clock](#the-database-clock) for the
`started_at <= now` comparison, so a retry or delayed execution does not become
claimable until its scheduled time, regardless of clock skew between hosts.

### Claiming branches (pipeline advancers)

A pipeline advancer claims a `branch` to move it to its next step. This is an
atomic conditional `UPDATE` (`Ductwork::BranchClaim`):

```sql
UPDATE ductwork_branches
   SET claimed_for_advancing_at = :now,
       claim_token              = :new_uuid,
       status                   = 'advancing'
 WHERE id = :id
   AND claimed_for_advancing_at = :previous_value
```

Only one advancer can transition the branch out of its previous
`claimed_for_advancing_at` value, so only one wins. The winner also generates a
fresh **claim token** (a UUID) and, inside the same transaction, opens the
two-phase advancement records (see [below](#two-phase-advancement-commit)).

## Claim (fencing) tokens

Atomic claiming alone is not enough. Consider: an advancer claims a branch,
stalls (GC pause, host freeze, network partition to the DB), is declared dead
by the reaper, the branch is released, a *second* advancer claims it and starts
working, and then the first advancer wakes up and tries to finish its now-stale
work. This is the classic distributed-systems "zombie writer" problem.

Ductwork solves it with **fencing tokens**, called *claim tokens* in the code.
Every successful branch claim writes a new random `claim_token`. The advancer
holds the token value it saw at claim time. Every state-mutating step of
advancement runs inside a fence:

```ruby
branch.with_claim_fence! do
  # ... mutate steps, transitions, create next steps/branches ...
end
```

`with_claim_fence!` re-reads the branch's current `claim_token` under a row
lock and compares it to the token the advancer is holding. If they differ (i.e.
the branch was reaped and re-claimed by someone else) it raises
`Ductwork::Branch::StaleClaimError` and the transaction rolls back. The stale
advancer's work is discarded harmlessly; the new owner is unaffected.

When an advancer finishes (or aborts) cleanly it calls `release!`, which is
itself token-guarded:

```sql
UPDATE ductwork_branches
   SET claimed_for_advancing_at = NULL,
       claim_token              = NULL,
       status                   = 'in_progress',
       last_advanced_at         = :now
 WHERE id = :id
   AND claim_token = :expected_token
   AND status      = 'advancing'
```

A stale process cannot release a branch it no longer owns, because its token no
longer matches.

## Heartbeats and orphan detection

Every supervisor process maintains a row in `ductwork_processes`
(`pid`, `machine_identifier`, `role`, `last_heartbeat_at`). On startup a
process *adopts or creates* its row
(`Ductwork::Process.adopt_or_create_current!`); while running it refreshes
`last_heartbeat_at` on every supervisor tick.

A process is considered **orphaned** when its `last_heartbeat_at` is older than
`Ductwork::Process::REAP_THRESHOLD` (currently **1 minute**). The host
identifier comes from `/etc/machine-id` (falling back to the hostname), so
`pid` collisions across machines cannot cause a healthy process on one host to
be mistaken for a dead one on another.

## The database clock

Heartbeat freshness and work-availability checks are time comparisons across
potentially many hosts. If each Ruby process used its own wall clock, NTP drift
could make healthy work look stale (premature reaping) or stale work look
healthy (work that never recovers).

`Ductwork::DatabaseClock` sidesteps this entirely by emitting SQL that resolves
"now" against the **database server's** clock: `clock_timestamp()` on Postgres,
`CURRENT_TIMESTAMP(6)` on MySQL, `julianday('now')` on SQLite, and the
equivalent on CockroachDB and Oracle. Every staleness predicate (reaper sweeps,
branch/job candidate selection, health checks) is expressed as a `DatabaseClock`
fragment, so the entire deployment shares a single authoritative clock
regardless of host time skew.

## The reaper

The reaper is what turns "a process died" into "its work runs again." Reaping
runs on every supervisor tick (`Ductwork::Process.reap_all!`) and also targets
specific PIDs when the process supervisor notices a dead forked child.

For each orphaned process row, `reap!` runs inside a single transaction that
first re-confirms staleness under a lock (so two supervisors can't double-reap)
and then:

1. **Abandons in-flight advancements**: every open `advancement` for that
   process is completed with error `Ductwork::ProcessCrash`, and its branch is
   `release!`d so a healthy advancer can pick it up again.
2. **Crashes in-flight executions**: every open `execution` for that process
   is closed via `crashed!`: the current attempt is finalized, a
   `process_crashed` result is recorded, and a **new execution + availability**
   is enqueued with `crash_count` incremented. The job becomes claimable again.
3. **Deletes the process row.**

If a row is gone before the transaction completes, the reaper treats it as
"already reaped by another supervisor" and moves on. Reaping is idempotent and
safe to run concurrently from multiple supervisors.

### Process restart vs. record reaping

These are two separate mechanisms working together:

- **Forking mode**: the `ProcessSupervisor` polls its children. A child whose
  heartbeat is stale is `SIGKILL`ed, its process record is reaped, and a fresh
  child is forked in its place.
- **Threaded mode**: the `ThreadSupervisor` restarts any worker thread that is
  no longer `alive?` and reports a heartbeat for the process as a whole.

In both cases the supervisor *also* runs `reap_all!`, so orphans from a
crashed *peer* host (whose supervisor can't restart anything) are still
recovered by any surviving supervisor.

## The execution commit guard

The reaper and a slow-but-alive worker can briefly race: the reaper crashes an
execution it believes is orphaned at almost the same moment the original worker
finishes and tries to commit. Ductwork closes this window with a guarded commit.

`succeeded!` and `errored!` commit only via a conditional update:

```sql
UPDATE ductwork_executions
   SET completed_at = :now
 WHERE id = :id
   AND completed_at IS NULL
   AND process_id   = :owner_process_id
```

If the reaper already closed the execution (or it was re-assigned), zero rows
update and Ductwork raises `Ductwork::Execution::CommitFailed`
("Reaper clobbered claimed job execution"). The worker discards the result and
loops; the re-enqueued execution created by the reaper is the one that counts.
This is the mechanism that makes "at-least-once" safe rather than
"sometimes-committed-twice."

## Two-phase advancement commit

Branch advancement can create downstream rows (new steps, new branches, jobs,
availabilities) and these must not be observable until the advancement that
produced them has actually committed. Ductwork uses paired **transition** and
**advancement** records:

- A `transition` represents the logical edge being traversed for a branch.
- An `advancement` represents one *attempt* by one process to traverse it,
  linked to that process.

When a branch is claimed, an open `advancement` is created (and any
pre-existing open advancement from a crashed prior attempt is failed with
`Ductwork::ProcessCrash`). All structural changes happen inside
`with_claim_fence!`, and the transition/advancement are stamped
`completed_at` in the same transaction that performs the work. A crash anywhere
in between rolls the transaction back; the branch is released (by the reaper or
the fence) and re-advanced from a consistent state, never a partially-applied
fan-out.

## Retries vs. crashes

Ductwork distinguishes two failure modes on an execution, tracked by two
separate counters:

| | `retry_count` | `crash_count` |
|---|---|---|
| Cause | The step raised an exception | The owning process died (reaped) |
| Counted against | `job_worker.max_retry` | Not capped; a crash is not the job's fault |
| Backoff | New execution scheduled `FAILED_EXECUTION_TIMEOUT` (10s) out | Re-enqueued immediately |
| Terminal result | Step marked `failed` → branch halts | Always retried |

This separation means an infrastructure failure (deploy, OOM, spot-instance
reclaim) never consumes a step's application-level retry budget.

## Manual recovery: `pipeline.revive!`

Automatic recovery handles process and host failure. It does **not**
automatically retry a pipeline that *halted* because a step exhausted its
application retries or hit a guardrail (invalid transition, fan-out limit,
unmatched condition). For those, recovery is an explicit operator or business
decision.

`pipeline.revive!` builds a new run from the halted one:

- Successful and in-progress branches/steps are duplicated as already-completed
  (no rework).
- Halted branches are revived: prior steps are copied as completed, and the
  failed/last step is re-enqueued or re-advanced.
- Pipeline-level context is optionally duplicated
  (`revive!(duplicate_context: true)`).

It raises `Ductwork::ReviveError` if the pipeline is not halted or has no prior
run. See [Error Handling](error-handling.html) for halting and `on_halt`.

## Configuration

| Setting | Default | Effect on durability |
|---|---|---|
| `Ductwork::Process::REAP_THRESHOLD` | 60s | How long after the last heartbeat a process is declared orphaned |
| `Execution::FAILED_EXECUTION_TIMEOUT` | 10s | Backoff before an *errored* execution becomes claimable again |
| `supervisor.polling_timeout` | n/a | How often heartbeats refresh and the reaper sweeps |
| `job_worker.max_retry` | n/a | Application retry budget before a step fails and the branch halts |
| `pipeline_advancer.max_retry` | n/a | Advancement retry budget before the branch halts (`advancer_retries_exhausted`) |
| `*.shutdown_timeout` | 20–30s | Grace period for in-flight work on a clean shutdown (see [Signal Handling](signal-handling.html)) |

{: .note }
Set `supervisor.polling_timeout` comfortably below `REAP_THRESHOLD` so a
healthy process always heartbeats several times within one reap window. The
process supervisor intentionally uses a slightly tighter threshold
(`REAP_THRESHOLD - 10s`) when judging its own children so it restarts them
before a peer supervisor would reap their records.

## Guarantees and your responsibilities

**Ductwork guarantees:**

- No work is lost when a process or host dies; every claimed unit is either
  committed or re-enqueued by the reaper.
- No work is silently executed twice *to completion* against stale state; the
  commit guard and claim fence reject zombie writers.
- Recovery does not require operator intervention for process or host failure.

**At-least-once, not exactly-once.** A step *can* run more than once: a worker
may finish the side effects of `execute` and then die before the guarded
commit, after which the reaper re-enqueues it. Therefore:

- **Make `execute` idempotent**, or guard external side effects with the
  idempotency key available within the step: `step.idempotency_key`
- **Return only JSON-serializable values**; the durable hand-off between steps
  is the persisted return value, not an in-memory object.
- **Keep steps reasonably short** so they fit inside shutdown windows and so a
  crash re-runs less work.

## Failure scenario walkthroughs

**A job worker host is terminated mid-`execute`.**
The execution row stays open with the dead `process_id`. Within `REAP_THRESHOLD`
a surviving supervisor's reaper calls `crashed!`, records a `process_crashed`
result, and enqueues a fresh execution with `crash_count + 1`. Another worker
claims it. If the original host briefly comes back and tries to commit, the
guarded `UPDATE` matches zero rows → `CommitFailed`, result discarded.

**An advancer freezes after claiming a branch, then resumes.**
The reaper abandons its open advancement (`Ductwork::ProcessCrash`) and
`release!`s the branch; a new advancer claims it with a new `claim_token`. When
the frozen advancer resumes and enters `with_claim_fence!`, the token mismatch
raises `StaleClaimError`, its transaction rolls back, and nothing is corrupted.

**Two workers race for the same job.**
Both attempt the atomic claim; the database serializes them. Exactly one
update affects a row. The loser observes zero rows updated, logs
"avoided race condition," and moves on.

**The database is briefly unreachable.**
No claims or commits succeed during the outage; there is no in-memory state to
lose. When connectivity returns, processes that missed heartbeats may be reaped
and their work re-enqueued; processes that re-adopt their (possibly reaped) row
continue. Work resumes from the last committed state.
