---
title: Concurrency
parent: Advanced
nav_order: 50
---

# Concurrency

Ductwork uses a multi-process, multi-threaded architecture designed for resilience, scalability, and efficient resource utilization. Understanding this model helps you optimize performance and troubleshoot issues effectively.

## Architecture Overview

Each `bin/ductwork` instance runs a **Supervisor Process** that manages child processes for your configured pipelines. The supervisor orchestrates:

- **One Pipeline Advancer Thread per Pipeline** - Creates a thread for each configured pipeline to advance pipelines through their steps
- **One Job Worker Process per Pipeline** - Each creates multiple threads to execute step jobs concurrently

This design isolates pipeline execution while enabling concurrent processing within each pipeline.

## Process Model

### Supervisor Process

The supervisor is the parent process responsible for:
- Launching and monitoring all child processes (advancers and workers)
- Detecting failures through heartbeat monitoring
- Automatically restarting crashed processes
- Coordinating graceful shutdown

**Failure recovery:** The supervisor monitors child process health through periodic heartbeats. If a child process fails to report a heartbeat within 5 minutes—indicating a crash or hang—the supervisor automatically spawns a replacement process to maintain pipeline availability.

### Pipeline Advancer Process

A single advancer process handles all configured pipelines by creating one thread per pipeline. Each thread:
- Monitors its pipeline's current stage
- Checks when all steps in a stage reach `advancing` status
- Transitions the pipeline to the next stage when ready
- Sleeps for `pipeline_advancer.polling_timeout` seconds when no work is available

### Job Worker Processes

Each configured pipeline gets its own dedicated job worker process. The process creates multiple threads (configured via `job_worker.count`) that:
- Query the database for available jobs
- Execute step logic
- Update job status atomically
- Sleep for `job_worker.polling_timeout` seconds when no jobs are available

**Work distribution:** Each worker thread independently queries for the oldest available job using an atomic `UPDATE...WHERE` query. This ensures:
- No job is processed by multiple threads simultaneously
- Work is distributed fairly across threads
- No external queue or coordinator is needed

## Thread Model

### Job Worker Threads

Worker threads operate in a continuous loop:

1. **Query for work** - Fetch the oldest available job from the database
2. **Claim the job** - Atomically update the job record to mark it as in-progress
3. **Execute** - Run the step's `#execute` method
4. **Update status** - Mark the job as complete or failed
5. **Repeat** - If no jobs are available, sleep for the configured polling timeout

This simple, database-backed coordination eliminates the need for complex distributed locking or external job queues.

### Pipeline Advancer Threads

Each pipeline's advancer thread:

1. **Query pipeline state** - Check if all steps in the current stage are complete
2. **Advance if ready** - When all steps reach `advancing` status, create the next stage's step records
3. **Sleep** - If the pipeline isn't ready to advance, sleep for the polling timeout
4. **Repeat** - Continue monitoring until the pipeline completes

## Database Connections

Worker threads check out connections from Rails' connection pool. **This is critical for configuration:**

```yaml
# config/database.yml
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 10 } %>
```

```yaml
# config/ductwork.yml
default: &default
  job_worker:
    worker_count: 10
  pipeline:
    worker_count: 10
```

**Recommendation:** Scale your connection pool size to match the number of worker threads. If you have 3 pipelines with 10 workers each and 10 pipeline advancers total, you'll need at least 40 database connections.

## Scaling Considerations

### Vertical Scaling (More Threads per Process)

Increase `job_worker.count` to process more jobs concurrently within each pipeline:

**Pros:**
- Simple configuration change
- Better resource utilization on larger machines
- Lower overhead than multiple processes

**Cons:**
- Requires more database connections
- Limited by single-process resources
- Ruby GIL may limit CPU-bound work

### Horizontal Scaling (Multiple Instances)

Run multiple `bin/ductwork` instances with different configuration files:

**Pros:**
- Better fault isolation
- Can run different pipelines on different machines
- Bypasses single-process limitations

**Cons:**
- More complex orchestration
- Higher operational overhead
- More database connections total

**Example:**
```bash
# High-priority pipelines on one instance
bin/ductwork -c config/ductwork.critical.yml

# Background pipelines on another
bin/ductwork -c config/ductwork.background.yml
```

## Performance Tips

- **Keep steps short** - Design steps to complete in seconds, or at least less than the shutdown timeouts
- **Use `expand` wisely** - Expanding to millions of steps can overwhelm the system
- **Monitor connection pool** - Watch for connection exhaustion as you scale threads
- **Tune polling timeouts** - Shorter timeouts increase responsiveness but add database load
- **Profile your steps** - Use logging and monitoring to identify slow steps
- **Consider batching** - Instead of expanding to 1,000,000 steps, batch into 10,000 steps that process 100 items each
