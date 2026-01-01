---
title: Configuration
parent: Pro
nav_order: 10
---

# Configuration

See the main [Configuration]({% link getting-started/configuration.md %}) guide for core settings and defaults.

Ductwork Pro extends the base configuration with additional options for performance tuning and observability.

## `pipeline_advancer.count`

The open-source version of Ductwork runs a single thread per pipeline class. Ductwork Pro lets you scale this up for high-throughput environments where pipeline advancement becomes a bottleneck—common when running large worker pools.

> ⚠️ **Caution:** Over-provisioning threads can increase database load and degrade performance. Start conservative and scale based on observed metrics.

**Default:** 3 (threads)

### Global thread count

```yaml
default: &default
  pipeline_advancer:
    count: 5
```

### Per-pipeline thread counts

For fine-grained control, specify counts by pipeline class:

```yaml
default: &default
  pipeline_advancer:
    count:
      default: 7
      MyPipelineA: 5
      MyPipelineB: 2
```

Pipelines without explicit configuration inherit the `default` value.


## `statsd=`

When set, ductwork will report metrics for the following internal events:

* `pipeline.triggered` - a COUNT metric reported each time a pipeline is triggered
* `pipeline.halted` - a COUNT metric reported each time a pipeline enters the "halted" state
* `pipeline.completed` - a COUNT metric reported when a pipeline finishes and enters the "completed" state
* `pipeline.runtime` - a TIMER metric type reported for the millisecond length of time an entire pipeline runs
* `job.enqueued` - a COUNT metric reported when a job is enqueued
* `job.completed` - a COUNT metric reported when a job completes; this includes successfully, erroring, or killed from a timeout
* `job.runtime` - a TIMER metric reported for the millisecond length of time that a job runs

This configuration is bespoke in that it must be set directly on the `Ductwork` top-level module as it takes an object. It can take any statsd object but it is recommended to use the `dogstatsd-ruby` [gem](https://github.com/datadog/dogstatsd-ruby).

```ruby
require "datadog/statsd"

Ductwork.statsd = -> { Datadog::Statsd.new("localhost", 8125) }
```
