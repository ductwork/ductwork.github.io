---
title: Signal Handling
parent: Advanced
nav_order: 40
---

# Signal Handling

## TTIN

Ductwork responds to the TTIN signal by dumping detailed backtraces for all child threads in the process—a powerful tool for diagnosing hung or stalled processes in production.

When you send a TTIN signal to a Ductwork process, it immediately outputs the complete call stack for each running thread. Each backtrace is prefixed with the thread's name in the format `ductwork.<role>_<id>`, making it easy to identify which worker or advancer is doing what.

**Usage:**

```bash
# Find the Ductwork process ID
ps aux | grep ductwork

# Send the TTIN signal
kill -TTIN <pid>
```

The output prints directly to STDOUT instead of the configured logger.

**Example Output:**

```
ductwork.job_worker_3
  /app/lib/some_operation.rb:42:in `process_data'
  /app/steps/my_step.rb:15:in `perform'
  ...

ductwork.pipeline_advancer_1
  /gems/ductwork/lib/pipeline_advancer.rb:28:in `advance'
  ...
```

## INT and TERM

Ductwork handles `INT` (Interrupt) and `TERM` (Terminate) signals identically, both triggering a coordinated graceful shutdown sequence across all processes. This ensures your pipelines shut down cleanly, giving in-flight jobs a chance to complete before the process exits.

### How It Works

When the supervisor receives an `INT` or `TERM` signal, it orchestrates a cascading shutdown:

1. **Signal Forwarding** - The supervisor immediately forwards the signal to all child processes—both the pipeline advancer and job worker processes.

1. **Job Worker Shutdown** - Job workers attempt a graceful shutdown by waiting for all active threads to complete their current work and join the parent process. Each thread has up to `job_worker.shutdown_timeout` seconds (default: 20) to finish. Once the timeout expires, any remaining threads are killed immediately to prevent hanging.

1. **Pipeline Advancer Shutdown** - The pipeline advancer attemps a graceful shutdown by waiting for all active threads to complete their current work and join the parent process. Each thread has up to `pipeline_advancer.shutdown_timeout` seconds (default: 20) to finish. Once the timeout expires, any remaining threads are killed immediately to prevent hanging.

1. **Supervisor Shutdown** - The supervisor waits for all child processes to exit cleanly, respecting the `supervisor.shutdown_timeout` value (default: 30 seconds). If any processes are still alive after the timeout, they receive a `SIGKILL` and terminate immediately.

### Shutdown Sequence

```
INT/TERM received
    ↓
Supervisor forwards signal to children
    ↓
Job workers wait for threads (up to job_worker.shutdown_timeout)
    ├─ Threads finish gracefully ✓
    └─ Or are killed after timeout ✗
Pipeline advancer waits for threads (up to pipeline_advancer.shutdown_timeout)
    ├─ Threads finish gracefully ✓
    └─ Or are killed after timeout ✗
    ↓
Supervisor waits for children (up to supervisor.shutdown_timeout)
    ├─ Children exit gracefully ✓
    └─ Or are killed after timeout ✗
    ↓
Supervisor exits
```

### Best Practices

- **Set appropriate timeouts:** Ensure `supervisor.shutdown_timeout` is larger than `job_worker.shutdown_timeout` to allow proper cascading.
- **Keep jobs short:** Design step jobs to complete quickly so they can finish within the shutdown window.
- **Monitor shutdown:** Use lifecycle hooks like `on_worker_stop` and `on_supervisor_stop` to track shutdown behavior and timing.

### Usage

```bash
# Graceful shutdown with INT
kill -INT <pid>

# Or with TERM
kill -TERM <pid>
```

The graceful shutdown ensures data integrity and prevents orphaned work, making it safe to deploy updates or scale down your Ductwork processes without losing in-progress pipeline work.
