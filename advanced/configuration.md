---
title: Configuration
parent: Advanced
nav_order: 10
---

# Configuration

Ductwork is configured through a configuration file. The default lives at `config/ductwork.yml` There is a CLI option (`-c` or `--config`) for specifying the configuration file's relative path. This allows you to have multiple instances of `ductwork` run different pipelines and be scaled differently.

Below are all the configuration values, descriptions, and their defaults that are available to be tuned:

## `database`

This configuration value determines the database to connect to in multi-database rails applications. This value is converted to a symbol and passed to ActiveRecord's `connect_to` method in the `Ductwork::Record` abstract parent class. If no value is configured then the `connect_to` method is not called. The default value for this configuration is `nil`.

If you do use this configuration, then you'll need to manually move the generated migration files to the `migration_paths` directory for that database and run migrations.

```yaml
default: &default
  database: pipeline_database

production:
  <<: *default
```

The default value is `nil`. As such, the method to connect to another database is not called if nothing is configured.

## `job_worker.count`

This configuration value sets the number of threads that are created for each running pipeline's job worker process. These threads are responsible for running the actual job for each step. The default is **5**.

```yaml
default: &default
  job_worker:
    worker_count: 10
```

**NOTE**: Be sure to properly scale the rails connection pool size so the number of threads and connections are equal.

## `job_worker.max_retry`

This configuration value sets the number of times a step's job can be retried before marking it as an error and halting the pipeline. The default is **3**.

```yaml
default: &default
  job_worker:
    max_retry: 3
```

## `job_worker.polling_timeout`

Once a job worker boots, it enters the main work loop. In the loop, the job worker will attempt to fetch an available job and execute it. If no job exists, the thread will sleep for a configured amount of time as to not crush the database with queries. The default is **1** with a unit of "seconds".

```yaml
default: &default
  job_worker:
    polling_timeout: 0.5 # seconds
```

## `job_worker.shutdown_timeout`

Once the job worker graceful shutdown sequence has started, all job worker threads attempt to join with the parent process. Ideally, job execution is short enough that it finishes before the timeout. Otherwise, the thread will be killed. This value should be less than the supervisor shutdown timeout to properly cascade timeouts. The default value is **20** (seconds).

```yaml
default: &default
  job_worker:
    shutdown_timeout: 25 # seconds
```

## `logger`

The `logger` configuration value is the only value that is not defined in the configuration file as it requires a logger object instance. It is recommended to configure this via an initializer or somewhere during rails boot. There is an included initializer that sets the logger to the `Rails.logger` if one is not set. While it logically flows to share the same logger for everything, it is recommended to use a separate logger for the running `ductwork` process. The default logger prints to `STDOUT`.

```ruby
# config/initializers/ductwork.rb
Ductwork.configuration.logger = Logger.new("ductwork.log")
```

## `pipeline.polling_timeout`

This configuration value is similar to the other polling timeout configurations. The pipeline advancer process will query for pipelines whose last step is completed and needs advanced. If no pipelines are returned or all have been advanced then the process will sleep for the configured amount of seconds. The default value is **1** (second).

```yaml
default: &default
  pipeline:
    polling_timeout: 5 # seconds
```

## `pipelines`

This configuration value sets the pipelines to run for the `ductwork` process. Running a pipeline means running a "pipeline advancer" process and a "job worker" process that creates threads. For more information on the concurrency models of `ductwork` see the [Concurrency page]({% link advanced/concurrency.md %}). The value for this configuration can be a list of pipeline class names or a wildcard. There is no default value for this configuration; it must be specified.

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

**NOTE**: Use with caution as this can eat up A LOT of resources if you have many defined pipelines!

## `supervisor.polling_timeout`

This configuration value follows the same pattern as the other polling timeout configuration values. The supervisor checks the heartbeat of its children processes then sleeps for the configured polling timeout in seconds. The default value is **1** (second).

```yaml
default: &default
  supervisor:
    polling_timeout: 5 # seconds
```

## `supervisor.shutdown_timeout`

This configuration value is similar to the other shutdown timeout configuration. Since this is the supervisor, this configuration value should be larger than all the other shutdown timeout configurations to ensure proper cascading. If the timeout is surpassed, the child processes: pipeline advancer and job worker are sent a `SIGKILL` signal and terminated immediately. The default value is **30** (seconds).

```yaml
default: &default
  supervisor:
    shutdown_timeout: 20 # seconds
```
