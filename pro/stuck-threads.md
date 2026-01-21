---
title: Stuck Threads
parent: Pro
nav_order: 35
---

# Stuck Threads

{: .highlight }
This feature requires [Ductwork Pro](https://getductwork.io/#pricing).

Ductwork Pro automatically detects and restarts threads that have stopped responding, keeping your pipelines running without manual intervention.

## How It Works

Every pipeline advancer and job worker thread updates a heartbeat timestamp as part of its work loop. The supervisor monitors these timestamps and restarts any thread whose heartbeat hasn't been updated within the timeout period.

| Component | Considered Stuck When |
|-----------|----------------------|
| Job worker | No job claimed or in-progress, and heartbeat exceeds timeout |
| Pipeline advancer | No pipeline claimed or advancing, and heartbeat exceeds timeout |

The default timeout is **5 minutes**. This value may become configurable in a future release.

---

## Supervisor Responsibilities

Which supervisor monitors heartbeats depends on your concurrency model:

- **Forking model:** Each pipeline advancer and job worker process supervises its own child threads.
- **Threaded model:** The top-level supervisor monitors all threads directly.

When a stuck thread is detected, the supervisor terminates it and spawns a replacement.

---

## Relationship to Step Timeouts

Stuck thread detection is separate from [Step Timeouts]({% link pro/step-timeout.md %}).

| Feature | What It Measures | When It Applies |
|---------|------------------|-----------------|
| Step Timeouts | Execution time of a single step | While a job is claimed and in-progress |
| Stuck Thread Detection | Thread responsiveness | When a thread is idle (no active work) |

Step timeouts ensure individual steps don't run too long. Stuck thread detection catches threads that have hung while waiting for work—for example, due to a deadlock or an unhandled exception in the polling loop.

{: .note }
There is currently no timeout for advancing a pipeline. This feature may be added in a future release.
