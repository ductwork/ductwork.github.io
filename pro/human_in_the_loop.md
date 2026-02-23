---
title: Human-in-the-Loop
parent: Pro
nav_order: 55
---

# Human-in-the-Loop

{: .highlight }
This feature requires [Ductwork Pro](https://getductwork.io/#pricing).

Ductwork Pro lets you pause a pipeline before a specific step and wait for human input before continuing. When a pipeline reaches a `dampen` transition, it enters an `dampened` state, holding its position until your application supplies the required input.

Paused pipelines don't consume worker threads while waiting. The pipeline advancer resumes them as soon as it is resumed.

---

## Why use human-in-the-loop?

The `dampen` transition lets you embed manual checkpoints directly into your pipeline definitions:

- **Approval workflows** - require a manager to sign off before a document is published
- **Financial authorizations** - pause before processing a large transaction until a reviewer confirms
- **Content moderation** - hold user-submitted content for manual review before publishing
- **Escalation handling** - route edge cases to a human before automated processing continues
- **Audit checkpoints** - enforce explicit sign-off before sensitive operations run

Unlike ad-hoc approaches (custom job queues, feature flags, or manual database checks), `dampen` keeps your workflow logic in one place—inside the pipeline definition itself.

---

## Declaring a dampen transition

Use `dampen(before: StepClass)` to insert a pause point before a specific step:

```ruby
define do |pipeline|
  pipeline.start(CreateDraftInvoice)
          .dampen(before: SendInvoice)
end
```

When the pipeline reaches the `dampen` transition, it pauses in the `dampened` state. `SendInvoice` will not run until input is provided by your application.

---

## What happens at a dampen transition

1. **Pipeline pauses** - the pipeline transitions to the `dampened` state
2. **Worker thread is released** - the thread is freed to process other work
3. **Pipeline waits indefinitely** - no timeout applies to the awaiting period itself

---

## Resuming

Call `#resume!` on the pipeline instance to resume it. The argument is optional. If omitted, the previous step's output payload is passed to the next step unchanged. If provided, it replaces that payload entirely:

```ruby
pipeline = InvoicePipeline.find(params[:pipeline_id])

# Resume with the previous step's output passed through unchanged
pipeline.resume!

# Resume with a custom payload passed to the dampened step's initializer
pipeline.resume!(input_args: { approved: true, reviewer_id: current_user.id })
```

The pipeline advancer picks it up and continues execution from where it left off.
