---
title: Testing
parent: Getting Started
nav_order: 50
---

# Testing

Ductwork provides custom RSpec matchers and some helpers to simplify pipeline testing.

## RSpec matchers

### `have_triggered_pipeline`

Use this matcher to verify that a single pipeline was triggered. It requires a block and takes one pipeline class as its argument.

```ruby
expect do
  MyService.call
end.to have_triggered_pipeline(MyPipeline)
```

#### Specifying Execution Count

You can assert that a pipeline was triggered a specific number of times:

```ruby
expect do
  MyService.call
end.to have_triggered_pipeline(MyPipeline).exactly(3).times
```

### `have_triggered_pipelines`

Use this matcher to verify that multiple pipelines were triggered. It takes multiple pipeline classes as arguments.

```ruby
expect do
  MyService.call
end.to have_triggered_pipelines(MyPipelineA, MyPipelineB, MyPipelineC)
```

**Note:** This matcher does not support count expectations, as specifying counts for multiple pipelines would be ambiguous.

---

## Creating steps

Because of Reasons, Ductwork steps are built in a non-standard way. This means you need to use the following to instantiate steps:

```ruby
let(:step) { FetchUserData.build_for_execution(pipeline_id, attrs) }
```

The first argument is the pipeline ID to which the step belongs followed by any input arguments.

---

## Creating pipelines

To make creating pipeline objects easier you can use the `pipeline_for` helper which only takes a class name:

```ruby
let(:pipeline) { pipeline_for(EnrichAllUsersDataPipeline) }
```

---

## Setting context

Likewise there is a helper to make setting global state easier:

```ruby
before do
  set_pipeline_context(pipeline, user_id: 1)
end
```

For convenience, you can use string or symbol keys to set context state.

---

## Pipeline validation

It is useful to know if your pipeline and step classes are valid before pushing to production. There is a top-level `Ductwork.validate!` method that will ensure your step classes have the correct methods and arity and that each pipeline is calling transitions correctly. It is recommended to call this method in a test:

```ruby
RSpec.describe "Ductwork" do
  specify "valid pipelines and steps" do
    Ductwork.validate!
  end
end
```
