---
title: Scaling
parent: Advanced
nav_order: 45
---

# Scaling

Ductwork is able to scale from a single `bin/ductwork` instance as well as running multiple `bin/ductwork` processes across many machines. This flexibility allows you to scale based on your needs.

## Vertical Scaling (More Threads per Process)

Increase `job_worker.count` to process more jobs concurrently within each pipeline:

**Pros:**
- Simple configuration change
- Better resource utilization on larger machines
- Lower overhead than multiple processes

**Cons:**
- Requires more database connections
- Limited by single-process resources
- Ruby GIL may limit CPU-bound work

## Horizontal Scaling (Multiple Instances)

Run multiple `bin/ductwork` instances with different configuration files:

**Pros:**
- Better fault isolation
- Can run different pipelines on different machines
- Bypasses single-process limitations

**Cons:**
- More complex orchestration
- Higher operational overhead
- More database connections total

**Example:**
```bash
# High-priority pipelines on one instance
bin/ductwork -c config/ductwork.critical.yml

# Low-priority pipelines on another
bin/ductwork -c config/ductwork.low.yml
```

## Database Connections

Worker threads check out connections from Rails' connection pool. Originally, it was recommended to properly calculate how large of a connection pool you would need. For many reasons, arriving at the perfect number can be difficult. [Current recommendation](https://island94.org/2024/09/secret-to-rails-database-connection-pool-size) is to set it to a large number and worry no more!

```yaml
# config/database.yml
production:
  pool: 100
```

**Be careful** that if you have a lot of pipelines configured for a single `bin/ductwork` process you may need to do a rough calculation to make sure a pool size of 100 is sufficient. For example, if you have 10 pipelines with 10 workers each and 10 pipeline advancers total, you'll need roughly 110 database connections.

## Performance Tips

- **Keep steps short** - Design steps to complete in seconds, or at least less than the shutdown timeouts
- **Monitor connection pool** - Watch for connection exhaustion as you scale threads
- **Tune polling timeouts** - Shorter timeouts increase responsiveness but add database load
- **Profile your steps** - Use logging and monitoring to identify slow steps
