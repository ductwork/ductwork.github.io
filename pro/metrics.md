---
title: Metrics
parent: Pro
nav_order: 60
---

# Metrics

Ductwork Pro can report internal events to a StatsD-compatible metrics backend, giving you visibility into pipeline and job performance in real time.

---

## Why instrument Ductwork?

Metrics help you answer operational questions:

- **Throughput** — How many pipelines are running? How many jobs are being processed?
- **Performance** — How long do pipelines and jobs take? Are there slowdowns?
- **Reliability** — How often do pipelines halt? Are timeouts firing frequently?
- **Capacity planning** — Is your worker pool keeping up with demand?

Combined with dashboards and alerting, these metrics form the foundation of production observability for your background processing.

---

## Available metrics


| Metric | Type | Description |
|--------|------|-------------|
| `pipeline.triggered` | count | Pipeline execution started |
| `pipeline.completed` | count | Pipeline finished successfully |
| `pipeline.halted` | count | Pipeline entered halted state (timeout, error, or manual halt) |
| `pipeline.runtime` | timer | Total pipeline duration from start to completion (ms) |
| `job.enqueued` | count | Job added to the queue |
| `job.completed` | count | Job finished—includes success, error, and timeout outcomes |
| `job.runtime` | timer | Job execution duration (ms) |

---

## Configuration

Assign a StatsD client to `Ductwork.statsd`. This must be set directly on the module since it accepts an object rather than a configuration value.

```ruby
require "datadog/statsd"

Ductwork.statsd = -> { Datadog::Statsd.new("localhost", 8125) }
```

The lambda ensures lazy initialization—the client is created when first needed rather than at boot time.

Any StatsD-compatible client works, but we recommend [dogstatsd-ruby](https://github.com/DataDog/dogstatsd-ruby) for its reliability and Datadog integration.

### Tagging and namespacing

If your StatsD client supports tags (like DogStatsD), you can configure a namespace and default tags:

```ruby
Ductwork.statsd = -> {
  Datadog::Statsd.new(
    "localhost",
    8125,
    namespace: "ductwork",
    tags: ["env:#{Rails.env}", "service:api"]
  )
}
```

This prefixes all metrics with `ductwork.` and attaches environment and service tags for filtering in your dashboards.

---

## Alerting suggestions

Consider setting alerts for:

- **High halt rate** — More than X% of pipelines halting may indicate a systemic issue
- **Runtime degradation** — Average pipeline or job runtime exceeding historical baselines
- **Queue backup** — `job.enqueued` significantly outpacing `job.completed` over time
- **Timeout spikes** — Sudden increase in `pipeline.halted` correlated with timeout-related errors
