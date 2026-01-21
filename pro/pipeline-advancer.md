---
title: Pipeline Advancer
parent: Pro
nav_order: 50
---

# Pipeline Advancer

{: .highlight }
This feature requires [Ductwork Pro](https://getductwork.io/#pricing).

The open-source version of Ductwork runs a single thread per pipeline class for advancing pipelines. Ductwork Pro lets you scale this for high-throughput environments where pipeline advancement becomes a bottleneck.

Advancer bottlenecks are most common with:

- Large worker pools that churn through jobs quickly
- Pipelines with many fast steps (sub-second execution times)
- Burst traffic patterns that create sudden pipeline backlogs

---

## Configure

The `pipeline_advancer.count` configuration sets the number of threads.

**Default:** 1 (threads)

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

> ⚠️ **Caution:** Over-provisioning threads can increase database load and degrade performance. Start conservative and scale based on observed metrics.
