---
title: Concurrency Models
parent: Architecture
nav_order: 20
---

# Concurrency Models

Ductwork supports two concurrency models to accommodate different runtime environments and deployment constraints.

| Model | Configuration | Best For |
|-------|---------------|----------|
| Forking | `forking: default` | MRI/CRuby (default) |
| Threaded | `forking: none` | JRuby, TruffleRuby, Windows, memory-constrained environments |

## Forking Model

The forking model is Ductwork's default. The supervisor forks child processes, each of which spawns threads based on your configuration.

### Configuration

No configuration is required since forking is the default. However, if you want to be explicit:

```yaml
# config/ductwork.yml
forking: default
```

### Process Hierarchy

```
bin/ductwork (supervisor)
├── pipeline advancer process
│   ├── thread for PipelineA
│   └── thread for PipelineB
├── job worker process for PipelineA
│   ├── worker thread 1
│   ├── worker thread 2
│   └── worker thread 3
└── job worker process for PipelineB
    ├── worker thread 1
    ├── worker thread 2
    └── worker thread 3
```

The supervisor creates one pipeline advancer process that spawns a thread for each configured pipeline class. It also forks separate job worker processes for each pipeline, and each of those processes spawns worker threads based on configuration (5 by default).

### Why Use Forking?

**True concurrency on MRI/CRuby.** Threads in MRI Ruby run under the Global VM Lock (GVL), which prevents true parallel execution. Separate processes bypass this limitation, giving you real concurrency.

**Process isolation.** If a pipeline encounters errors or crashes, it doesn't affect other pipelines. Each job worker process operates independently.

### Trade-offs

Forked processes copy memory, so memory usage scales with the number of pipelines. Strategies for managing this include running multiple `bin/ductwork` processes across machines or switching to the threaded model. See [Scaling]({% link advanced/scaling.md %}) for more details.

## Threaded Model

The threaded model runs everything in a single process using only threads. The supervisor spawns pipeline advancer threads and job worker threads directly, without forking.

### Configuration

```yaml
# config/ductwork.yml
forking: none
```

See [Configuration]({% link getting-started/configuration.md %}) for additional options.

### Process Hierarchy

```
bin/ductwork (supervisor)
├── pipeline advancer thread for PipelineA
├── pipeline advancer thread for PipelineB
├── worker thread 1 for PipelineA
├── worker thread 2 for PipelineA
├── worker thread 3 for PipelineA
├── worker thread 1 for PipelineB
├── worker thread 2 for PipelineB
└── worker thread 3 for PipelineB
```

### Why Use Threaded?

**JRuby and TruffleRuby compatibility.** These runtimes don't support `fork`. The threaded model lets you run Ductwork on JVMs and alternative Ruby implementations.

**Windows support.** Windows lacks native `fork` support. Use the threaded model when deploying to Windows environments.

**Lower memory footprint.** Threads share memory with the parent process. In memory-constrained environments (small containers, edge deployments), this can be a significant advantage over forking.

**Simpler debugging.** A single process is easier to profile, trace, and attach debuggers to.

### Trade-offs

On MRI/CRuby, threads don't provide true parallelism due to the GVL. CPU-bound work won't benefit from multiple threads in the same way it would from multiple processes. However, if your pipelines are I/O-bound (database queries, HTTP requests, file operations), the threaded model still provides effective concurrency.

Additionally, a crash in one thread can potentially affect the entire process since there's no process-level isolation between pipelines.
