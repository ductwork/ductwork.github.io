---
title: Running Pipelines
parent: Getting Started
nav_order: 30
---

# Running Pipelines

## Starting the Ductwork Process

If you recall, after running the Rails generator you'll have a new binstub at `bin/ductwork`. This executable launches the supervisor process, which orchestrates your entire pipeline infrastructure—forking a pipeline advancer and job worker for each configured pipeline, with the job worker creating multiple threads to handle concurrent work.

The supervisor continuously monitors child processes via heartbeat checks. When a child process misses heartbeats for 5 minutes—indicating a crash or hang—the supervisor immediately spawns a replacement process to ensure minimal pipeline execution interruption.

### Basic Usage

```bash
bin/ductwork
```

### Custom Configuration

Use the `-c` or `--config` flag to specify a custom YAML configuration file. This lets you run multiple Ductwork instances with different pipeline configurations and scaling settings:

```bash
bin/ductwork -c config/ductwork.yml
bin/ductwork --config config/ductwork.0.yml
```

**Pro tip:** Run separate Ductwork instances with different configurations to isolate high-priority pipelines or scale specific workloads independently.

## Triggering Pipelines from Your Code

With Ductwork running, you can trigger pipelines from anywhere in your Rails application. The `.trigger` method enqueues the pipeline and returns a `Ductwork::Pipeline` instance that you can query for status, progress, or results.

### Example: Rake Task

```ruby
task enrich_user_data: :environment do
  pipeline = EnrichAllUsersDataPipeline.trigger(7)
  puts "Pipeline #{pipeline.id} started!"
end
```

### Example: Controller Action

```ruby
class DataEnrichmentController < ApplicationController
  def create
    days_outdated = params[:days_outdated] || 7
    pipeline = EnrichAllUsersDataPipeline.trigger(days_outdated)

    render json: {
      pipeline_id: pipeline.id,
      status: pipeline.status
    }
  end
end
```

### Example: Scheduled Task

```ruby
# config/schedule.rb (using whenever gem)
every 1.day, at: '3:00 am' do
  runner "EnrichAllUsersDataPipeline.trigger(7)"
end
```

## Monitoring Your Pipeline

The returned pipeline instance gives you real-time access to pipeline state:

```ruby
pipeline = EnrichAllUsersDataPipeline.trigger(7)

pipeline.id          #=> "123"
pipeline.status      #=> "in_progress", "completed", "failed", etc.
pipeline.steps       #=> Collection of all steps in the pipeline
pipeline.created_at  #=> Timestamp
```

---

That's it! Your pipelines are now running and processing work.
