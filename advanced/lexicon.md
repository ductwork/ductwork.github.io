---
title: Lexicon
parent: Advanced
nav_order: 10
---

# Lexicon

This page defines the core concepts and components. Understanding these terms will hopefully help you properly design and scale your pipelines.

## Pipeline

A **pipeline** is a complete workflow definition consisting of steps and the transitions between them. Pipelines are represented as Ruby classes that inherit from `Ductwork::Pipeline` and live in `app/pipelines`.

Each pipeline defines:

- The sequence of steps to execute
- How data flows between steps
- How work is distributed (sequential, parallel, or fan-out patterns)

## Branch

A **branch** is a path in a pipeline. Pipelines can have many branches. Branches are created when the `divide`, `expand`, or `divert` transitions are called, and branches are reduced when the complementary `combine`, `collapse`, or `converge` transitions are called.

## Step

A **step** is a checkpoint within a pipeline. Steps are Ruby objects that inherit from `Ductwork::Step` and implement two methods:

- `initialize` - Accepts arguments from the trigger or previous step's return value
- `execute` - Performs the work and returns data for the next step

**Key principle:** Each step's return value becomes the next step's input. Return values must be JSON-serializable.

## Job

A **job** is the representation of work for a step. A job may have multiple executions depending on if an error was raised or not. When a pipeline runs, Ductwork creates job records in the database for each step that needs to execute. Jobs track:

- Which step class to run
- What arguments to pass to the step
- Execution status (`pending`, `in_progress`, `completed`, `failed`)
- Retry attempts
- Error messages

## Job Worker

A **job worker** is a process responsible for executing jobs. Each configured pipeline gets its own dedicated job worker process, which creates multiple threads (configured via `job_worker.count`) to execute jobs concurrently.

**Responsibilities:**

- Query the database for available jobs
- Claim jobs atomically to prevent duplicate processing
- Execute the job's step logic
- Update job status and handle failures
- Retry failed jobs up to the configured limit

Job workers use an `UPDATE...WHERE` query to atomically claim work, ensuring no job is processed by multiple threads simultaneously.

## Pipeline Advancer

A **pipeline advancer** is a process responsible for moving pipelines forward through their stages. A single advancer process handles all configured pipelines by creating one thread per pipeline.

**Responsibilities:**
- Monitor pipeline progress by checking step completion status
- Advance pipelines to the next stage when all current steps are complete
- Create new job records for the next steps in the workflow
- Handle pipeline completion and error states

The advancer uses database queries to determine when a pipeline is ready to advance. When all steps in the current stage reach `advancing` status, the advancer transitions the pipeline to the next stage by creating job records for subsequent steps.

## Supervisor

The **supervisor** is the parent process that orchestrates all Ductwork processes. Each `bin/ductwork` instance runs one supervisor, which manages:
- One pipeline advancer process
- One job worker process per configured pipeline

The supervisor monitors child processes by checking periodic heartbeats. If a child process fails to report a heartbeat within 5 minutes, the supervisor automatically spawns a replacement to maintain pipeline availability.

**Process hierarchy:**
```
supervisor (parent)
├── pipeline advancer
│   └── thread per pipeline
├── job worker (Pipeline A)
│   └── multiple worker threads
├── job worker (Pipeline B)
│   └── multiple worker threads
└── job worker (Pipeline C)
    └── multiple worker threads
```

## Relationships

Understanding how these concepts relate:

1. A **pipeline** defines the workflow structure
2. When triggered, the pipeline creates a **job** for each step
3. **Job workers** execute those jobs by running step logic
4. The **pipeline advancer** monitors completion and creates steps and jobs for subsequent steps
5. The **supervisor** ensures all processes stay running and healthy
