---
title: Testing
parent: Getting Started
nav_order: 50
---

# Testing

Ductwork provides custom RSpec matchers to simplify pipeline testing.

## `have_triggered_pipeline`

Use this matcher to verify that a single pipeline was triggered. It requires a block and takes one pipeline class as its argument.

```ruby
expect do
  MyService.call
end.to have_triggered_pipeline(MyPipeline)
```

### Specifying Execution Count

You can assert that a pipeline was triggered a specific number of times:

```ruby
expect do
  MyService.call
end.to have_triggered_pipeline(MyPipeline).exactly(3).times
```

## `have_triggered_pipelines`

Use this matcher to verify that multiple pipelines were triggered. It takes multiple pipeline classes as arguments.

```ruby
expect do
  MyService.call
end.to have_triggered_pipelines(MyPipelineA, MyPipelineB, MyPipelineC)
```

**Note:** This matcher does not support count expectations, as specifying counts for multiple pipelines would be ambiguous.
