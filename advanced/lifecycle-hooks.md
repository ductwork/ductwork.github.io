---
title: Lifecycle Hooks
parent: Advanced
nav_order: 30
---

# Lifecycle Hooks

Ductwork emits events at specific points during startup and graceful shutdown. You can register actions for these events using methods on the top-level `Ductwork` module. Each method can be called multiple times to register multiple actions. Due to the Rails boot sequence, it's best to define these hooks in a Rails initializer.

## Available Hooks

Each hook method has the format `on_<target>_<event>` and takes a block.

### `on_supervisor_start`

Fires when the supervisor process starts. This event occurs once during boot.

**Example:**
```ruby
# config/initializers/ductwork.rb
Ductwork.on_supervisor_start do
  Ductwork.logger.info "Ductwork supervisor started"
end
```

### `on_supervisor_stop`

Fires when the supervisor has stopped or killed all of its child processes. This event occurs once during shutdown.

**Example:**
```ruby
Ductwork.on_supervisor_stop do
  Ductwork.logger.info "Ductwork supervisor stopped"
end
```

### `on_advancer_start`

Fires when the pipeline advancer process starts. This event occurs once during boot.

**Example:**
```ruby
Ductwork.on_advancer_start do
  Ductwork.logger.info "Pipeline advancer process started"
end
```

### `on_advancer_stop`

Fires when the pipeline advancer process stops. This is the last action called before the process exits and occurs once during shutdown.

**Example:**
```ruby
Ductwork.on_advancer_stop do
  Ductwork.logger.info "Pipeline advancer process stopped"
end
```

### `on_worker_start`

Fires for each job worker thread created by the job worker runner process. This event is called once per thread at the beginning of thread creation, occurring as many times as you have configured worker threads.

**Example:**
```ruby
Ductwork.on_worker_start do
  Ductwork.logger.info "Job worker thread started"
end
```

### `on_worker_stop`

Fires for each job worker thread that is still alive when the shutdown sequence begins. This is the last action that runs before the thread exits.

**Example:**
```ruby
Ductwork.on_worker_stop do
  Ductwork.logger.info "Job worker thread stopped"
end
```

## Usage Example

Here's a complete example of registering lifecycle hooks in a Rails initializer:

```ruby
# config/initializers/ductwork.rb
Ductwork.on_supervisor_start do
  MetricsReporter.increment("ductwork.supervisor.start")
end

Ductwork.on_worker_start do
  # Set up thread-local resources
  Thread.current[:started_at] = Time.current
end

Ductwork.on_worker_stop do
  # Clean up thread-local resources
  duration = Time.current - Thread.current[:started_at]
  Ductwork.logger.info "Worker thread ran for #{duration} seconds"
end

Ductwork.on_supervisor_stop do
  MetricsReporter.increment("ductwork.supervisor.stop")
  # Clean up resources, flush metrics, etc.
end
```

## Hook Execution Order

**During startup:**
1. `on_supervisor_start`
2. `on_advancer_start`
3. `on_worker_start` (for each worker thread)

**During shutdown:**
1. `on_worker_stop` (for each active worker thread)
2. `on_advancer_stop`
3. `on_supervisor_stop`
