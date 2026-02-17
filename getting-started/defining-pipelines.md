---
title: Defining Pipelines
parent: Getting Started
nav_order: 20
---

# Defining Pipelines

A pipeline consists of steps and the transitions between them. Ductwork provides a simple DSL for defining these steps and transitions in a clear, readable way.

## Creating a Pipeline Class

Pipeline classes **must** live under `app/pipelines` to be discovered correctly. While class names don't require a "Pipeline" suffix, adding one can help prevent naming collisions depending on your conventions.

Create a pipeline by inheriting from `Ductwork::Pipeline`:

```ruby
# app/pipelines/enrich_all_users_data.rb
class EnrichAllUsersData < Ductwork::Pipeline
end

# Or with a suffix for clarity
# app/pipelines/enrich_all_users_data_pipeline.rb
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
end
```

Each pipeline class automatically gets a default scope, allowing you to query for pipelines of that specific type:

```ruby
:001> EnrichAllUsersDataPipeline.in_progress.to_sql
#=> "SELECT \"ductwork_pipelines\".* FROM \"ductwork_pipelines\"
#    WHERE \"ductwork_pipelines\".\"klass\" = 'EnrichAllUsersDataPipeline'
#    AND \"ductwork_pipelines\".\"status\" = 'in_progress'"
```

## Defining Steps

Steps are Ruby objects that inherit from `Ductwork::Step` with a simple interface. They must live under `app/steps` and implement an `#execute` instance method that takes no arguments.

The initializer's parameters depend on either:
- The arguments passed when the pipeline is triggered (for the first step)
- The return value from the previous step (for subsequent steps)

This simple interface makes steps highly testable without external dependencies. Like pipelines, step class names don't require a "Step" suffix, though it may help with organization.

**Example Step:**

```ruby
# app/steps/query_users_requiring_enrichment.rb
class QueryUsersRequiringEnrichment < Ductwork::Step
  def initialize(days_outdated)
    @days_outdated = days_outdated
  end

  def execute
    ids = User.where("data_last_refreshed_at < ?", @days_outdated.days.ago).ids
    Rails.logger.info("Enriching #{ids.length} users' data")

    # Return the collection of IDs to pass to the next step
    ids
  end
end
```

**Design philosophy:** Steps represent checkpoints in your pipeline and should complete quickly. Break down long-running jobs into smaller work units by chaining steps together and passing data between them. The examples throughout this page demonstrate this approach.

## Understanding Transitions

Transitions connect steps together. The key principle: **the return value of one step becomes the input to the next step.** This means you need to align the initializer's arity between connected steps.

**Important considerations:**
- Align parameter counts between connected steps
- Use the splat operator (`*args`) if you prefer treating everything as an array
- Return values must be JSON-serializable

The Ductwork DSL uses a fluent interface pattern, enabling clean method chaining.

## Transition Types

### `start` - Define the First Step

The `start` transition defines your pipeline's entry point. This step receives the arguments passed to the `.trigger` method (see [Running Pipelines]({% link getting-started/running-pipelines.md %})).

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
  end
end
```

### `expand` - Fan Out to Multiple Steps

The `expand` transition acts like a "foreach" loop. It takes the return value from the previous step (which must be a collection) and creates a new step instance for each element. All expanded steps may run concurrently, depending on your job worker scaling.

The return value must be JSON-serializable and respond to `#each`.

**Syntax:** `expand(to: StepClass)`

```ruby
# Remember in our step class that we returned an array of IDs. Each scalar ID
# will get passed to a new `LoadUserData` step.
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
  end
end
```

### `divide` - Split Into Parallel Branches

The `divide` transition passes a copy of the return value to multiple steps simultaneously. Each step receives identical input and may run concurrently.

**Syntax:** `divide(to: [StepClass1, StepClass2, ...])`

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
            .divide(to: [FetchDataFromSourceA, FetchDataFromSourceB])
  end
end
```

**Block syntax:** The `divide` transition accepts an optional block, yielding a branch for each step. This allows you to chain different transitions onto each branch independently:

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(StepA)
            .divide(to: [StepB, StepC, StepD]) do |branch1, branch2, branch3|
              branch1.chain(to: StepD)
              branch2.expand(to: StepE)
              branch3.chain(to: StepF).chain(to: StepG)
            end
  end
end
```

### `combine` - Merge Branches Back Together

The `combine` transition is the inverse of `divide`. It merges multiple branches into a single step, combining the return values from all previous steps into an array. The combined step waits for all previous steps to complete before starting.

**Syntax:** `combine(into: StepClass)`

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
            .divide(to: [FetchDataFromSourceA, FetchDataFromSourceB])
            .combine(into: CollateUserData)
  end
end
```

**Block syntax:** For improved readability, you can call `combine` on any branch and pass the other branches as arguments:

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(StepA).divide(to: [StepB, StepC, StepD]) do |branch1, branch2, branch3|
      branch1.combine(branch2, branch3, into: StepE)
    end
  end
end
```

**Note:** Calling `combine` on a pipeline that hasn't been divided raises a `Ductwork::DSL::DefinitionBuilder::CombineError`.

### `chain` - Connect Steps Sequentially

The `chain` transition is the simplest transition. It connects two steps sequentially—the second step only runs if the first succeeds. There's no concurrency with chaining.

**Syntax:** `chain(to: StepClass)`

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
            .divide(to: [FetchDataFromSourceA, FetchDataFromSourceB])
            .combine(into: CollateUserData)
            .chain(to: UpdateUserData)
  end
end
```

### `collapse` - Gather Expanded Steps

The `collapse` transition is the inverse of `expand`. It gathers all steps from an expanded branch back into a single step, combining their return values into an array. Like `combine`, the collapsed step waits for all expanded steps to complete.

**Syntax:** `collapse(into: StepClass)`

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
            .divide(to: [FetchDataFromSourceA, FetchDataFromSourceB])
            .combine(into: CollateUserData)
            .chain(to: UpdateUserData)
            .collapse(into: ReportUserEnrichmentSuccess)
  end
end
```

**Note:** Calling `collapse` on a pipeline that hasn't been expanded raises a `Ductwork::DSL::DefinitionBuilder::CollapseError`.

---

Next, we're ready to run the `ductwork` processes.
