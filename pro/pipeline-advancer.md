---
title: Pipeline Advancer
parent: Pro
nav_order: 50
---

## `pipeline_advancer.count`

The open-source version of Ductwork runs a single thread per pipeline class. Ductwork Pro lets you scale this up for high-throughput environments where pipeline advancement becomes a bottleneck—common when running large worker pools.

> ⚠️ **Caution:** Over-provisioning threads can increase database load and degrade performance. Start conservative and scale based on observed metrics.

**Default:** 3 (threads)

### Global thread count

```yaml
default: &default
  pipeline_advancer:
    count: 5
```

### Per-pipeline thread counts

For fine-grained control, specify counts by pipeline class:

```yaml
default: &default
  pipeline_advancer:
    count:
      default: 7
      MyPipelineA: 5
      MyPipelineB: 2
```

Pipelines without explicit configuration inherit the `default` value.
