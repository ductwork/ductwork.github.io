---
title: Enhanced Durability
parent: Pro
nav_order: 55
---

# Enhanced Durability

{: .highlight }
This feature requires [Ductwork Pro](https://getductwork.io/#pricing).

Enhanced durability ensures your pipelines never get stuck due to ungraceful shutdowns. When a worker thread is forcibly terminated, Ductwork Pro automatically requeues the interrupted job for execution, eliminating data loss and preventing pipelines from entering unrecoverable states.

## How It Works

### Graceful Shutdown Behavior

When a shutdown signal is received, Ductwork supervisors enter a grace period, allowing in-progress jobs time to complete naturally. You can configure this timeout for each component:

| Component | Configuration | Link |
|-----------|---------------|------|
| Job workers | `job_worker.shutdown_timeout` | [link](https://docs.getductwork.io/getting-started/configuration.html#job_workershutdown_timeout)
| Pipeline advancers | `pipeline_advancer.shutdown_timeout` | [link](https://docs.getductwork.io/getting-started/configuration.html#pipeline_advancershutdown_timeout)

### When Jobs Don't Complete in Time

**Open-source behavior**: If a job is still running when the configured grace period expires, the thread is killed. This can leave jobs in an inconsistent state—partially completed, with no record of what happened.

**Ductwork Pro behavior**: When the grace period expires, Pro handles the situation gracefully:

1. The supervisor terminates the thread
1. The current execution is marked as "timed out"
1. The job's retry count increments
1. If retries remain, the job is automatically requeued
1. If retries are exhausted, the pipeline halts cleanly

## Best Practices
While enhanced durability protects against data loss, shorter jobs recover faster from interruptions. Design jobs to be as concise as possible. Break large operations into smaller, discrete, `chain`-ed steps when feasible.
