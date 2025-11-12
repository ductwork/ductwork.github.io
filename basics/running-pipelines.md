---
title: Running Pipelines
parent: Basics
nav_order: 30
---

# Running Pipelines

After running the rails generator you will have a new binstub at `bin/ductwork`. This executable starts the supervisor which then forks a pipeline advancer and step worker (which then creates threads) for each configured pipeline.

The CLI has a single option specified by either `-c` or `--config`. It takes a path to the YAML configuration file you want to load for the running instance of `ductwork`.

```bash
❯ bin/ductwork -c config/ductwork.yml
```

---

All that's left now is to call the pipeline in your code! Triggering a pipeline returns a `Ductwork::Pipeline` instance.

```ruby
rake :enrich_user_data do
  EnrichAllUsersDataPipeline.trigger(7)
end
```
