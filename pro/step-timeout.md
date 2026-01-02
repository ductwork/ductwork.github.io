---
title: Step Timeout
parent: Pro
nav_order: 20
---

# Step Timeout

Ductwork Pro lets you enforce time limits on individual step executions. When a step exceeds its timeout, the pipeline halts automatically—preventing runaway processes from blocking your workers indefinitely.

## Why use timeouts?

Timeouts protect your system from:

- **External service hangs** — API calls that never return
- **Infinite loops** — bugs that cause steps to run forever
- **Resource exhaustion** — long-running steps consuming worker threads
- **Cascading delays** — one slow step backing up your entire queue

They're especially valuable in production environments where predictable execution times matter.

## Declaring timeouts

Add a `timeout` option to any step in your pipeline definition. Values can be integers (seconds) or `ActiveSupport::Duration` objects:

```ruby
define do |pipeline|
  pipeline.start(FetchData, timeout: 1.minute)
          .chain(ProcessData, timeout: 60)
end
```

Both formats work identically—use whichever reads more clearly for your use case.

> ⚠️ **Validation:** Timeout values must be positive. Zero or negative values raise an `ArgumentError` at definition time.

## What happens when a step times out

When a step exceeds its timeout, Ductwork Pro triggers the following sequence:

1. **Thread termination** — The worker thread executing the step is killed immediately
2. **Pipeline halt** — The pipeline transitions to the `halted` state
3. **Thread restart** — A fresh worker thread is spawned to replace the terminated one

The halted pipeline can be inspected, retried, or cleaned up depending on your error-handling strategy.

### Monitoring timeouts

If you've configured [Metrics]({% link pro/metrics.md %}), timeouts are reflected in:

- `pipeline.halted` — incremented when the pipeline enters the halted state
- `job.completed` — incremented (with the `result` tag set to `killed`)

For more information you can check the logs for `"Pipeline halted"` line which will contain the pipeline ID. You can also see the halted pipeline in the dashboard.
