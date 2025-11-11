---
title: Defining Pipelines
parent: Basics
nav_order: 20
---

# Defining Pipelines

A Pipeline is made up of Steps and the Transitions between them. Ductwork ships with a simple DSL to make defining these Steps and Transitions easy.

To start a Pipeline definition you first need a Pipeline class. This class **must** live under `app/pipelines` for it to be discovered correctly. Class names do not need to be prefixed with "Pipeline" but it may prevent naming collisions with other classes depending on existing naming conventions. A Pipeline class simply inherits from `Ductwork::Pipeline`.


```ruby
# app/pipelines/enrich_all_users_data.rb
class EnrichAllUsersData < Ductwork::Pipeline
end

# app/pipelines/enrich_all_users_data_pipeline.rb
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
end
```

## Steps

Steps are simply ruby POROs (Plain Old Ruby Object) with a specific interface. They must live under `app/steps`, similar to Pipelines. The interface requires an `#execute` instance method that doesn't take any arguments. The arity of the initializer depends on the arguments passed when triggered or the previous Step's return value. Because of the simple interface, Steps are easily testable without needing external dependencies (unless your logic requires it). Similar to Pipelines, Step class names do not need a "Step" suffix, but again, it may provide naming benefits. Assuming one of the Pipelines above exists, continue with an example:

```ruby
# app/steps/query_users_requiring_enrichment.rb`
class QueryUsersRequiringEnrichment
  def initialize(days_outdated)
    @days_outdated = days_outdated
  end

  def execute
    ids = User.where("data_last_refreshed_at < ?", @days_outdated.days.ago).ids
    Rails.logger.info("Enriching #{ids.length} users' data")
    # We specifically return the collection of IDs because we want this
    # as the argument to the next Step.
    ids
  end
end
```

Conceptually, Steps represent a checkpoint in your Pipeline. They are intended to run in a short amount of time. The philosophy is to break down long running jobs into smaller work units by pipelining Steps and passing return data. The running example through this page will illustrate this.

## Transitions

Now that we've covered Steps, let's go over Transitions. The most important thing to remember with transitions is that the return value of the previous Step is the input to the next Step. This requires you to align arity of the initializer between each Step. It's not as hard as it may sound when you have good tests in place. The other option is to use the splat operator for all arguments and treat everything as an array. One other consideration is that argument types must be JSON serializable. The `ductwork` DSL implements a fluent interface pattern which allows for method chaining.

### Start

Our first transition, is kinda not a transition at all. It simply defines the first Step to start the Pipeline with. As you'll see on the [Triggering Pipelines] page, the first Step gets the arguments that are passed to the `.trigger` method.

```ruby
class EnrichAllUsersDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
  end
end
```

### Expanding to a Step

The `expand` transition is basically a "foreach". The return value of the Step `QueryUsersRequiringEnrichment` is multiplexed and new Steps are created for each element in the collection. All of these steps may run concurrently, but in actuality it matters on how you scale the job workers. Any collection that is JSON serializable and responds to `#each` can be returned.

The `expand` Transition has a single keyword argument `to` that takes a single Step class. In our example below:

```ruby
class EnrichUserDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
  end
end
```

### Dividing to Multiple Steps

The `divide` transition passes a copy of the return value to each Step class in the Transition argument. This argument is a keyword argument, similar to `expand`, with a single `to` keyword but takes an array of Step classes instead of a scalar. All of these steps may also run concurrently.

```ruby
class EnrichUserDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
            .divide(to: [FetchDataFromSourceA, FetchDataFromSourceB])
  end
end
```

The `divide` Transition is interesting in that it can also take a block. When taking a block `divide` will yield a Branch for each Step in the arguments. This allows you to further chain Transitions onto each Branch separately.

```ruby
class EnrichUserDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(StepA)
            .divide(to: [StepB, StepC, StepD]) do |branch1, branch2, branch3|
              branch1.chain(StepD)
              branch2.expand(to: StepE)
              branch3.chain(StepF).chain(StepG)
              # ...
            end
  end
end
```

### Combining Steps into a Single Step

The `combine` Transition is the inverse of the `divide` Transition. It takes multiple Branches and merges them into a single Step. The return values from each previous Step is combined together in an array and passed to the next Step. The "combined" Step waits for all previous Steps to finish before starting. Combine will raise a `Ductwork::DSL::DefinitionBuilder::CombineError` if you call `combine` on a pipeline that hasn't been divided.

```ruby
class EnrichUserDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
            .divide(to: [FetchDataFromSourceA, FetchDataFromSourceB])
            .combine(into: CollateUserData)
  end
end
```

The block syntax of the `combine` Transition can be a bit easier to read. You call `combine` on any of the branches and pass any as the first argument.

```ruby
class EnrichUserDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(A).divide(to: [B, C, D]) do |branch1, branch2, branch3|
      branch1.combine(branch2, branch3, into: StepE)
    end
  end
end
```

### Chaining Steps

The most basic Transition is `chain`. This simply connects two Steps together, executing the latter one if and only if the previous Step succeeds. There is no concurrency with chaining.

```ruby
class EnrichUserDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
            .divide(to: [FetchDataFromSourceA, FetchDataFromSourceB])
            .combine(into: CollateUserData)
            .chain(UpdateUserData)
  end
end
```

### Collapsing into a Single Step

The `collapse` Transition is the inverse of the `expand` Transition. It takes an expanded Branch and collapses all the Steps into a new Step. Similar to `combine` it merges together all the arguments into an array and passes that to the next Step. Collapse will raise a `Ductwork::DSL::DefinitionBuilder::CollapseError` if the pipeline is not expanded.

```ruby
class EnrichUserDataPipeline < Ductwork::Pipeline
  define do |pipeline|
    pipeline.start(QueryUsersRequiringEnrichment)
            .expand(to: LoadUserData)
            .divide(to: [FetchDataFromSourceA, FetchDataFromSourceB])
            .combine(into: CollateUserData)
            .chain(UpdateUserData)
            .collapse(into: ReportUserEnrichmentSuccess)
  end
end
```

---

Now we're finally ready to trigger the pipelines to run!
