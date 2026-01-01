---
title: Supervisor
parent: Architecture
nav_order: 30
---

# Supervisor

The supervisor is the orchestrator of your Ductwork infrastructure. As the parent process for each `bin/ductwork` instance, it launches and monitors all child processes, ensuring your pipelines continue running even when individual processes fail.

## Overview

When you run `bin/ductwork`, you start a supervisor process that:
- Launches one pipeline advancer process
- Launches one job worker process for each configured pipeline
- Monitors all child processes through heartbeat checks
- Automatically restarts failed or hung processes
- Coordinates graceful shutdown across all processes

The supervisor acts as the resilience layer, making Ductwork pipelines fault-tolerant without manual intervention.

## Process Hierarchy

Each Ductwork instance creates the following process tree:

```
supervisor (bin/ductwork)
├── pipeline advancer
│   └── thread for Pipeline A
│   └── thread for Pipeline B
│   └── thread for Pipeline C
├── job worker (Pipeline A)
│   └── worker thread 1
│   └── worker thread 2
│   └── ...
├── job worker (Pipeline B)
│   └── worker thread 1
│   └── worker thread 2
│   └── ...
└── job worker (Pipeline C)
    └── worker thread 1
    └── worker thread 2
    └── ...
```

## Responsibilities

### Process Lifecycle Management

The supervisor manages the complete lifecycle of child processes:

**Startup:**
1. Read configuration from YAML file
2. Fork the pipeline advancer process
3. Fork one job worker process per configured pipeline
4. Register signal handlers for graceful shutdown
5. Enter monitoring loop

**Monitoring:**
- Check heartbeats from each child process
- Track process health and uptime
- Detect crashes or hangs
- Log process status changes

**Recovery:**
- Automatically restart failed processes
- Maintain pipeline availability during failures
- Preserve pipeline state through process restarts

**Shutdown:**
- Forward shutdown signals to all children
- Wait for graceful shutdown with timeout
- Terminate unresponsive processes
- Clean up resources and exit

### Heartbeat Monitoring

The supervisor continuously monitors child process health through periodic heartbeats. Each child process reports its status at regular intervals, confirming it's alive and processing work.

**Detection:** If a child process fails to report a heartbeat within 5 minutes—indicating a crash, hang, or deadlock—the supervisor detects the failure.

**Recovery:** The supervisor immediately spawns a replacement process to restore full pipeline capacity. The new process picks up where the previous one left off, resuming work on pending jobs.

**Why 5 minutes?** This timeout balances quick failure detection with tolerance for legitimately slow operations. Steps should typically complete in seconds, but this buffer accounts for temporarily degraded performance without false positives.

## Configuration

The supervisor's behavior is controlled through `config/ductwork.yml`:

### `pipelines`

Specifies which pipelines to run. The supervisor creates child processes based on this configuration.

```yaml
default: &default
  pipelines:
    - EnrichUserDataPipeline
    - ProcessOrdersPipeline
```

Or use the wildcard to run all defined pipelines:

```yaml
default: &default
  pipelines: "*"
```

**Note:** The supervisor creates one advancer and one job worker per pipeline listed here.

### `supervisor.polling_timeout`

How long (in seconds) the supervisor sleeps between heartbeat checks.

**Default:** 1 second

```yaml
default: &default
  supervisor:
    polling_timeout: 5
```

**Tuning:** Shorter intervals provide faster failure detection but increase CPU usage. Longer intervals reduce overhead but delay failure detection. The default (1 second) works well for most applications.

### `supervisor.shutdown_timeout`

Maximum time (in seconds) to wait for child processes to shut down gracefully. After this timeout, remaining processes receive `SIGKILL` and terminate immediately.

**Default:** 30 seconds

```yaml
default: &default
  supervisor:
    shutdown_timeout: 45
```

**Important:** This value should be larger than `job_worker.shutdown_timeout` to allow proper cascading. If the supervisor timeout is too short, workers won't have time to finish their shutdown sequence.

**Recommended values:**
- `job_worker.shutdown_timeout`: 20 seconds
- `supervisor.shutdown_timeout`: 30 seconds (gives 10 seconds buffer)

## Signal Handling

The supervisor responds to Unix signals for control and debugging:

### TERM and INT - Graceful Shutdown

Triggers the graceful shutdown sequence:

1. Supervisor forwards signal to all child processes
2. Child processes begin their shutdown sequences
3. Supervisor waits up to `supervisor.shutdown_timeout` seconds
4. Processes still alive after timeout are killed with `SIGKILL`
5. Supervisor exits

```bash
# Send TERM signal
kill -TERM <supervisor_pid>

# Or INT signal (both behave identically)
kill -INT <supervisor_pid>
```

See [Signal Handling]({% link advanced/signal-handling.md %}) for detailed shutdown behavior.

### TTIN - Thread Backtrace Dump

Requests thread backtraces from all child processes for debugging hung or slow processes.

```bash
kill -TTIN <supervisor_pid>
```

The supervisor forwards this signal to all children, which dump their thread backtraces to the configured logger. This is invaluable for diagnosing performance issues or deadlocks in production.

See [TTIN Signal Handling]({% link advanced/signal-handling.md %}#ttin-signal-handling) for details.

## Lifecycle Hooks

Register actions to run when the supervisor starts or stops:

```ruby
# config/initializers/ductwork.rb

Ductwork.on_supervisor_start do
  Rails.logger.info "Ductwork supervisor starting"
  # Initialize monitoring, notify deployment tracking, etc.
end

Ductwork.on_supervisor_stop do
  Rails.logger.info "Ductwork supervisor shutting down"
  # Flush metrics, notify monitoring systems, etc.
end
```

These hooks run once per supervisor lifecycle—at the very beginning of startup and the very end of shutdown. Use them for initialization, cleanup, or integration with external systems.

See [Lifecycle Hooks]({% link advanced/lifecycle-hooks.md %}) for all available hooks.

## Monitoring

Track supervisor health and behavior by monitoring:

### Process Metrics
- Supervisor uptime
- Number of child process restarts
- Child process spawn rate
- Failed startup attempts

### Resource Usage
- Supervisor CPU and memory usage
- Total memory across all child processes
- Open file descriptors
- Database connection count

### Heartbeat Status
- Time since last heartbeat from each child
- Heartbeat check frequency
- Missed heartbeat count

### Shutdown Behavior
- Time to complete graceful shutdown
- Number of processes killed after timeout
- Shutdown success rate

## Running Multiple Supervisors

You can run multiple `bin/ductwork` instances to isolate pipelines or scale horizontally:

### Isolate Critical Pipelines

```bash
# Critical pipelines with dedicated resources
bin/ductwork -c config/ductwork.critical.yml

# Background pipelines on separate instance
bin/ductwork -c config/ductwork.background.yml
```

```yaml
# config/ductwork.critical.yml
production:
  pipelines:
    - ProcessPaymentsPipeline
    - SendNotificationsPipeline
  job_worker:
    worker_count: 20

# config/ductwork.background.yml
production:
  pipelines:
    - GenerateReportsPipeline
    - CleanupDataPipeline
  job_worker:
    worker_count: 5
```

### Scale Across Machines

Run separate supervisors on different servers for horizontal scaling:

```bash
# Server 1 - Handle user-facing pipelines
bin/ductwork -c config/ductwork.user_facing.yml

# Server 2 - Handle batch processing pipelines
bin/ductwork -c config/ductwork.batch.yml
```

**Benefits:**
- Fault isolation (one failing pipeline doesn't affect others)
- Resource allocation (dedicate CPU/memory to specific pipelines)
- Independent scaling (scale critical pipelines without scaling everything)
- Deployment flexibility (deploy changes to specific pipeline groups)

**Considerations:**
- More operational complexity
- Higher total resource usage (overhead per supervisor)
- Need coordination for monitoring across instances

## Process Management

Integrate Ductwork with your process manager:

### systemd

```ini
# /etc/systemd/system/ductwork.service
[Unit]
Description=Ductwork Pipeline Supervisor
After=network.target postgresql.service

[Service]
Type=simple
User=deploy
WorkingDirectory=/var/www/myapp
ExecStart=/var/www/myapp/bin/ductwork -c config/ductwork.yml
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Docker

```dockerfile
# Dockerfile
CMD ["bin/ductwork", "-c", "config/ductwork.production.yml"]
```

### Kubernetes

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ductwork
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: ductwork
        image: myapp:latest
        command: ["bin/ductwork"]
        args: ["-c", "config/ductwork.yml"]
```

The supervisor's resilient design makes it suitable for containerized environments and orchestration platforms.
