---
title: Home
layout: home
nav_order: 10
---

# Ductwork Documentation

**Durable workflows for Ruby**

Ductwork is a durable execution worflow orchestration engine and framework. You define a workflow as a series of steps; Ductwork runs them for you, on processes and threads it supervises inside your own application, and persists every step and every state transition to your relational database *before* acting on it. When a worker, a host, or the database itself goes away mid-flight, the work isn't lost. It resumes from the last committed state, automatically, with no message broker and no operator intervention.

## What Makes Ductwork Different

**It runs your workflows; it isn't just a way to describe them.** Ductwork supervises its own worker and advancer processes alongside your Rails app. You `trigger` a pipeline; Ductwork claims, executes, and advances each step. There is no separate scheduler or runtime to stand up.

**Your database is the source of truth.** Work is rows, not in-memory messages or broker queues. A triggered pipeline writes durable run, branch, step, and job records. No work exists only inside a running process, so a crash can't lose it.

**Automatic recovery from process and host failure.** Every worker heartbeats. When one stops, a reaper re-enqueues its in-flight work and a healthy worker picks it up. Fencing tokens and guarded commits ensure a recovered "zombie" process can never corrupt the work that moved on without it.

**At-least-once execution you can reason about.** Every step is guaranteed to run; infrastructure failures (OOM, spot reclaim, deploy) never consume a step's application retry budget. Make steps idempotent (Ductwork gives you `Step#idempotency_key`) and the durability contract is simple and explicit.

**No new infrastructure.** No Kafka, no Temporal server, no separate worker tier to operate. If you have a Rails app and a relational database, you have everything Ductwork needs.

## An Example Workflow

Each step below is a checkpoint. Ductwork persists its result before advancing, so a failure at any point resumes from there, not from the top.

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)           # Begin with a single step
            .expand(to: LoadUserData)                       # Fan out to process each user
            .divide(to: [FetchDataFromSourceA,              # Split into parallel branches
                         FetchDataFromSourceB])
            .combine(into: CollateUserData)                 # Bring branches back together
            .chain(to: UpdateUserData)                      # Sequential processing
            .divert(to: { success: NotifyUser,              # Conditional branching
                          otherwise: FlagForReview })
            .converge(into: FinalizeUserRecord)             # Merge conditional branches
            .collapse(into: ReportUserEnrichmentSuccess)    # Final aggregation
  end
end

# Trigger from anywhere in your Rails app, then walk away.
# Ductwork runs every step to completion across process and host failures.
EnrichAllUsersDataPipeline.trigger(days_outdated: 7)
```

A step is just a class. Because Ductwork may re-run a step after a crash, the durability contract is explicit: make side effects idempotent, and return JSON-serializable values. The persisted return value *is* the hand-off to the next step.

```ruby
class UpdateUserData < Ductwork::Step
  def execute(user_data)
    # `step.idempotency_key` is stable across retries and crash re-runs,
    # so external side effects can be made safely repeatable.
    ExternalCrm.upsert(user_data, key: step.idempotency_key)

    { user_id: user_data["id"], status: "updated" }  # durable hand-off to the next step
  end
end
```

## Core Concepts

- **[Getting Started]({% link getting-started/start-here.md %})** - Install Ductwork and run your first pipeline
- **[Defining Pipelines]({% link getting-started/defining-pipelines.md %})** - The DSL for connecting steps and managing workflow
- **[Durability]({% link advanced/durability.md %})** - How Ductwork persists, claims, fences, and recovers every unit of work
- **[Concurrency]({% link architecture/concurrency.md %})** - Ductwork's multi-process, multi-threaded execution model

## When to Use Ductwork

Ductwork is built for workflows where *finishing* matters: work that spans multiple steps and external systems and must survive deploys and failures:

- **Multi-step integrations** - Coordinate calls across services and APIs where a partial failure must resume, not restart
- **Data enrichment pipelines** - Fetch, transform, and enrich records from multiple sources with durable checkpoints between stages
- **ETL processes** - Extract, transform, and load with crash recovery instead of re-running the whole batch
- **Batch operations** - Process large datasets in concurrent chunks where each chunk's progress is persisted
- **Report generation** - Aggregate data from multiple sources into a final artifact that only emits once

If your "workflow" is a single background job, a plain job queue is enough. Reach for Ductwork when the work has multiple stages and losing progress is expensive.

## Ready to Start?

Jump into the [Installation Guide]({% link getting-started/start-here.md %}) and run your first durable pipeline in minutes, or read [how durability works]({% link advanced/durability.md %}) first.

Have questions? [Open an issue on GitHub](https://github.com/ductwork/ductwork/issues) or upgrade to Pro for custom support.
