---
title: Getting Started
parent: Basics
nav_order: 10
---

# Getting Started

Ductwork is designed for you to be up-and-running in only a few commands.

## Installation

Currently, `ductwork` must be installed in a rails application. In the future, it may be possible to run `ductwork` in any ruby project.

To install `ductwork` simply add it to your `Gemfile`:

```ruby
# Gemfile
gem "ductwork"
```

## Rails Generator

To help you get started there is a rails generator that creates necessary files for you: `bin/ductwork` binstub, configuration file at `config/ductwork.yml`, and all necessary migrations. Run it with:

```bash
❯ bin/rails generate ductwork:install
```

## Running Migrations
After that, the only thing left to do is run migrations via:

```bash
❯ bin/rails db:migrate
```

### Multi-Database Support

If you are connecting to multiple databases in your rails application you can configure which database to use with the `database` configuration. For more information see the [Configuration section]({% link advanced/configuration.md %}). Then, you'll need to copy over the migrations that were generated to the `migration_paths` configuration for that database before migrating.

---

Once that is completed you are ready to [define pipelines]({% link basics/defining-pipelines.md %}) and [run the `ductwork` process]({% link basics/running-pipelines.md %}).
