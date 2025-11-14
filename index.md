---
title: Home
layout: home
nav_order: 10
---

# Ductwork Documentation

**Build powerful data pipelines with Ruby—no complexity required.**

Ductwork is a pipeline framework designed for developers who want to get things done. Build sophisticated multi-stage workflows using familiar Ruby patterns and an intuitive DSL. No complicated object models, no separate infrastructure—just elegant Ruby code that scales.

## What Makes Ductwork Different

**Simple by design.** Define pipelines with a fluent DSL that reads like English. Connect steps, fan out work, merge results—all with natural Ruby syntax.

**Runs where your app runs.** No message brokers, no separate worker infrastructure. Ductwork runs alongside your Rails application using processes and threads you control.

**Built for resilience.** Automatic process recovery, graceful shutdowns, and configurable retries keep your pipelines running through failures.

**Scales with you.** Start with a single process and scale to hundreds of concurrent workers. Adjust thread counts, tune timeouts, and isolate critical pipelines—all through simple YAML configuration.

## Quick Example

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment) # Begin with a single step
            .expand(to: LoadUserData) # Fan out to process each user
            .divide(to: [FetchDataFromSourceA, FetchDataFromSourceB]) # Split into parallel branches
            .combine(into: CollateUserData) # Bring branches back together
            .chain(UpdateUserData) # Sequential processing
            .collapse(into: ReportUserEnrichmentSuccess) # Final aggregation
  end
end

# Trigger from anywhere in your Rails app
EnrichAllUsersDataPipeline.trigger(days_outdated: 7)
```

## Core Concepts

- **[Getting Started]({% link getting-started/start-here.md %})** - Install Ductwork and run your first pipeline
- **[Defining Pipelines]({% link getting-started/defining-pipelines.md %})** - Learn the DSL for connecting steps and managing workflow
- **[Configuration]({% link getting-started/configuration.md %})** - Tune workers, timeouts, and scaling for your needs
- **[Concurrency]({% link architecture/concurrency.md %})** - Understand Ductwork's multi-process, multi-threaded architecture

## When to Use Ductwork

Ductwork excels at multi-stage data processing workflows:

- **Data enrichment pipelines** - Fetch, transform, and enrich records from multiple sources
- **ETL processes** - Extract, transform, and load data with fault tolerance
- **Batch operations** - Process large datasets in manageable, concurrent chunks
- **Multi-step integrations** - Coordinate complex workflows across services and APIs
- **Report generation** - Aggregate data from multiple sources into comprehensive reports

## Ready to Start?

Jump into the [Installation Guide]({% link getting-started/start-here.md %}) and build your first pipeline in minutes.

Have questions? [Open an issue on GitHub](https://github.com/ductwork/ductwork/issues) or upgrade to Pro for custom support.
