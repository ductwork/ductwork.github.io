---
title: Error Handling
parent: Advanced
nav_order: 20
---

# Error Handling

When a step exceeds the maximum number of retries, the pipeline halts. You can specify a class to handle this situation using the `on_halt` DSL method.

Pass your halt handler class to the `on_halt` method in your pipeline definition:
```ruby
class EnrichUserData < Ductwork::Pipeline
  define do |pipeline|
    pipeline.on_halt(PageOnCallEngineer)
  end
end
```

## Interface

Your halt handler class must implement the following interface:

* `initialize(error)` - The initializer receives a single argument: the error instance from the final failed step.

* `execute` - An instance method that performs the halt handling logic (similar to pipeline steps).

## Organization

While halt handler classes can live anywhere in your application, placing them in `app/steps` is recommended since they share a similar interface with pipeline steps.
