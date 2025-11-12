---
title: Configuration
parent: Advanced
nav_order: 10
---

# Configuration

Ductwork is configured through a YAML configuration file. The default configuration file is located at `config/ductwork.yml`. You can specify a different configuration file using the CLI option `-c` or `--config` with a relative path. This allows you to run multiple instances of Ductwork with different pipelines and scaling configurations.

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

Configures the logger instance used by Ductwork. This is the only configuration value not defined in the YAML file, as it requires a logger object. Configure this in an initializer or during Rails boot. An included initializer sets the logger to `Rails.logger` if none is specified. While sharing the Rails logger is convenient, using a separate logger for the Ductwork process is recommended.

**Default:** Logger writing to `STDOUT`

```ruby
# config/initializers/ductwork.rb
Ductwork.configuration.logger = Logger.new("ductwork.log")
```

## `pipeline.polling_timeout`

Configures how long (in seconds) the pipeline advancer process sleeps when no pipelines need advancing. The process queries for pipelines whose last step is completed and advances them if found.

**Default:** 1 (second)

```yaml
default: &default
  pipeline:
    polling_timeout: 5 # seconds
```

## `pipelines`

Specifies which pipelines to run for the Ductwork process. Running a pipeline means starting both a pipeline advancer process and a job worker process with multiple threads. For more information on Ductwork's concurrency model, see the [Concurrency page]({% link advanced/concurrency.md %}).

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
