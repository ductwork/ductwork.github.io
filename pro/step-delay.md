---
title: Step Delay
parent: Pro
nav_order: 30
---

# Step Delay

{: .highlight }
This feature requires [Ductwork Pro](https://getductwork.io/#pricing).

Ductwork Pro lets you schedule steps to execute after a specified waiting period. When a pipeline reaches a delayed step, it pauses for the configured duration before the step begins execution.

Delayed steps don't block worker threads while waiting. The pipeline advancer periodically checks for steps that are ready to run, making this an efficient way to handle time-based logic.

---

## Why use delays?

Delays let you build time-aware workflows directly into your pipeline definitions:

- **Onboarding sequences** — send a welcome email immediately, then check profile completion a day later
- **Retry backoff** — wait before retrying a failed external service call
- **Rate limiting** — space out API calls to stay within third-party limits
- **Scheduled follow-ups** — remind users who haven't completed an action
- **Cooling-off periods** — enforce waiting periods before processing sensitive operations

Unlike scheduling jobs with external tools, delays keep your workflow logic in one place—inside the pipeline definition itself.

---

## Delaring delays

Add a `delay` option to any step in your pipeline definition. Values can be integers (seconds) or `ActiveSupport::Duration` objects:

```ruby
define do |pipeline|
  pipeline.start(SendWelcomeEmail, delay: 1.hour)
          .chain(CheckProfileComplete, delay: 3_600)
end
```

> ⚠️ **Validation:** Delay values must be positive. Zero or negative values raise an `ArgumentError` at definition time.

---

## Combining delays with other features

**Delays + Timeouts:** A step can have both a delay and a timeout. The timeout window begins when the step starts executing, not when it becomes eligible:

```ruby
pipeline.chain(CallExternalApi, delay: 5.minutes, timeout: 30.seconds)
```

**Delays + Conditional logic:** Delays work naturally with conditional branching—schedule different follow-up timing based on prior step outcomes.
