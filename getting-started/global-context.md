---
title: Global Context
parent: Getting Started
nav_order: 45
---

# Global Context

Ductwork provides a key-value store that persists across all steps in a pipeline. Unlike return values, which flow from one step to the next, global context is accessible from any step at any point in the pipeline's execution.

---

## Why use global context?

Return value passing works well for linear data flow, but some information needs to be available everywhere:

- **User or request identifiers** — set once at the start, referenced throughout
- **Configuration fetched early** — lookup results needed by multiple downstream steps
- **Sidecar data** — necessary values that don't quite make sense as data flow
- **Cross-cutting concerns** — tracing IDs, feature flags, or tenant context

Global context keeps this data accessible without threading it through every step's return value.

---

## Basic usage

Access global context through the `context` object, which is in-scope in every step:

```ruby
# Write a value
context.set("user_id", 123)

# Read it back
context.get("user_id")  #=> 123
```

Values can be any JSON serializable Ruby object: strings, numbers, hashes, or arrays.

---

## Write-once by default

To encourage disciplined use of shared state, context keys are **write-once by default**. Attempting to overwrite an existing key raises an error:

```ruby
context.set("user_id", 1)
context.set("user_id", 2)  #=> raises Ductwork::Context::OverwriteError
```

This prevents accidental overwrites and makes it easier to reason about where values come from.

### Explicit overwrites

When you genuinely need to update a value, pass `overwrite: true`:

```ruby
context.set("user_id", 1)
context.set("user_id", 2, overwrite: true)  # succeeds
context.get("user_id")  #=> 2
```

Use this sparingly. If you find yourself overwriting frequently, consider whether the data model fits better as step return values or a dedicated accumulator pattern.

---

## Atomicity and concurrency

Reads and writes to global context are **atomic** ie. each operation completes fully before another can begin. This prevents torn reads or partial writes.

However, atomicity doesn't make concurrent access safe in all cases:

```ruby
# Step A and Step B run concurrently
# Both try to set "status" at the same time

# Step A
context.set("status", "processing", overwrite: true)

# Step B
context.set("status", "validating", overwrite: true)

# Final value depends on execution order!
```

> ⚠️ **Caution:** Avoid writing to the same key from concurrent steps. Even with atomic operations, the final value depends on execution order. If concurrent steps need to share state, use separate keys or coordinate through step dependencies.

---

## Example: pipeline-wide tracing

A common pattern is setting a trace ID at the start of a pipeline for observability:

```ruby
class InitializeTracing < Ductwork::Step
  def execute
    trace_id = SecureRandom.uuid
    context.set("trace_id", trace_id)

    # trace_id is now available in all subsequent steps
  end
end

class ProcessOrder < Ductwork::Step
  def execute
    trace_id = context.get("trace_id")
    Rails.logger.info("[#{trace_id}] Processing order #{order.id}")

    # ... process order
  end
end
```
