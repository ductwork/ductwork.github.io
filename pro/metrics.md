---
title: Metrics
parent: Pro
nav_order: 60
---

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
