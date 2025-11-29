---
title: Configuration
parent: Pro
nav_order: 10
---

# Configuration

See main [Configuration]({% link getting-started/configuration.md %}) for main configuration values and defaults.

Ductwork Pro contains a few more configuration settings as outlined below:

## `pipeline_advancer.count`

This value is the number of pipeline advancer threads to create for each pipeline.

**Default:** 3 (threads)

```yaml
default: &default
  pipeline_advancer:
    count: 5
```

You can also specify thread count by pipeline.

```yaml
default: &default
  pipeline_advancer:
    count:
      default: 7
      MyPipelineA: 5
      MyPipelineB: 2
```
