---
title: Configuration
parent: Getting Started
nav_order: 40
---

# Configuration

Ductwork is configured through a YAML configuration file. The default configuration file is located at `config/ductwork.yml`. You can specify a different configuration file using the CLI option `-c` or `--config` with a relative path. This allows you to run multiple instances of Ductwork with different pipelines and scaling configurations.

You can also set configuration values directly in code, say in an initializer, but it is less recommended:
```ruby
# config/initializers/ductwork.rb
Ductwork.configuration.job_worker_max_retry = 10
```

Below are all available configuration values with their descriptions and defaults.

## `database`

Specifies which database to connect to in multi-database Rails applications. This value is converted to a symbol and passed to ActiveRecord's `connect_to` method in the `Ductwork::Record` abstract parent class. If left unset, the `connect_to` method is not called and the default database is used.

**Default:** `nil`

```yaml
default: &default
  database: pipeline_database

production:
  <<: *default
```

**Note:** If you use this configuration, you'll need to manually move the generated migration files to the `migration_paths` directory for that database and run migrations.

## `job_worker.count`

Sets the number of threads created for each running pipeline's job worker process. These threads execute the actual job for each step.

**Default:** 5

```yaml
default: &default
  job_worker:
    worker_count: 10
```

**Important:** Scale your Rails connection pool size to match the number of threads to ensure adequate database connections.

## `job_worker.max_retry`

Sets the maximum number of times a step's job can be retried before marking it as an error and halting the pipeline.

**Default:** 3

```yaml
default: &default
  job_worker:
    max_retry: 3
```

## `job_worker.polling_timeout`

Configures how long (in seconds) a job worker thread sleeps when no jobs are available. This prevents excessive database queries while waiting for work.

**Default:** 1 (second)

```yaml
default: &default
  job_worker:
    polling_timeout: 0.5 # seconds
```

## `job_worker.shutdown_timeout`

Sets the maximum time (in seconds) to wait for job worker threads to complete during graceful shutdown. Threads that don't finish within this timeout are killed. This value should be less than the supervisor shutdown timeout to ensure proper cascading.

**Default:** 20 (seconds)

```yaml
default: &default
  job_worker:
    shutdown_timeout: 25 # seconds
```

## `logger`

Configures the logger instance used by Ductwork. This is the only configuration value not defined directly in the YAML file, however, see below logger configurations below for more options. Configure this in an initializer or during Rails boot. An included Rails engine sets the `logger` to `Rails.logger` ONLY if it hasn't been set yet.

While it can be argued that an application triggering pipelines should log to its rails logger, for the ductwork process it makes sense to use its own logger. It's recommended to log to STDOUT and let your process manager handle it. Sharing the Rails logger can be convenient, but it can clutter output. It's recommended to use a separate logger.

**Default:** Logger writing to `STDOUT`

```ruby
# config/initializers/ductwork.rb
Ductwork.configuration.logger = Logger.new("ductwork.log")
```

## `logger.level`

Sets the log level.

**Default:** 1 (`Logger::INFO`)

```yaml
default: &default
  logger:
    level: 0 # Logger::DEBUG
```

**NOTE**: This will only change the logger level for the running `bin/ductwork` process. This is by design as to avoid accidentally changing the log level for an entires Rails application.

## `logger.source`

Instead of setting the `logger` variable directly, which isn't recommended unless necessary, you can choose between the default logger printing to STDOUT or the Rails logger.

**Default:** `"default"` (`Logger` instance that prints to STDOUT)

```yaml
default: &default
  logger:
    source: rails # sets to `Rails.logger`, do this with caution!
```

 **NOTE**: This will only change the logger level for the running `bin/ductwork` process.

## `pipeline_advancer.polling_timeout`

Configures how long (in seconds) the pipeline advancer process sleeps when no pipelines need advancing. The process queries for pipelines whose last step is completed and advances them if found.

**Default:** 1 (second)

```yaml
default: &default
  pipeline:
    polling_timeout: 5 # seconds
```

## `pipeline_advancer.shutdown_timeout`

Sets the maximum time (in seconds) to wait for pipeline advancer threads to complete during graceful shutdown. Threads that don't finish within this timeout are killed. This value should be less than the supervisor shutdown timeout to ensure proper cascading.

**Default:** 20 (seconds)

```yaml
default: &default
  pipeline_advancer:
    shutdown_timeout: 25 # seconds
```

## `pipeline_advancer.steps.max_depth`

Sets the maximum cardinality of steps in a single branch. This configuration only applies to the `expand` and `divide` transitions as the number of steps can increase while transitioning. If the maximum depth value is exceeded during pipeline advancement, the pipeline will be halted as it cannot continue.

**Default:** -1 (unlimited)

```yaml
default: &default
  pipeline_advancer:
    steps:
      max_depth: 1_000_000
```

You can also set the max step depth for a specific pipeline or even a specific step in a pipeline. When using more specific syntax you can still set the library-level default with the `max_depth.default` key.

```yaml
default: &default
  pipeline_advancer:
    steps:
      max_depth:
        default: 1_000_000
        MyPipelineA: 10_000
        MyPipelineB:
          default: 42
          MyStepA: 5_000
```

## `pipelines`

Specifies which pipelines to run for the Ductwork process. Running a pipeline means starting both a pipeline advancer process and a job worker process with multiple threads. For more information on Ductwork's concurrency model, see the [Concurrency page]({% link architecture/concurrency.md %}).

This configuration accepts either a list of pipeline class names or a wildcard. **This configuration is required and has no default value.**

```yaml
default: &default
  pipelines:
    - MyPipelineA
    - MyPipelineB
```

Use the wildcard `"*"` to run all defined pipelines:

```yaml
default: &default
  pipelines: "*"
```

**Warning:** Use the wildcard with caution, as it can consume significant resources if you have many defined pipelines.

## `supervisor.polling_timeout`

Configures how long (in seconds) the supervisor process sleeps between heartbeat checks of its child processes (pipeline advancer and job worker).

**Default:** 1 (second)

```yaml
default: &default
  supervisor:
    polling_timeout: 5 # seconds
```

## `supervisor.shutdown_timeout`

Sets the maximum time (in seconds) to wait for child processes to shut down gracefully. This value should be larger than all other shutdown timeout configurations to ensure proper cascading. If the timeout is exceeded, child processes (pipeline advancer and job worker) receive a `SIGKILL` signal and terminate immediately. Be sure to set this value less than the max timeout your cloud provider may have. For example, Heroku waits 30 seconds before sending SIGKILL to processes on the dynos.

**Default:** 30 (seconds)

```yaml
default: &default
  supervisor:
    shutdown_timeout: 20 # seconds
```
