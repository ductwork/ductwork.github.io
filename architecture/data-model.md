---
title: Data Model
parent: Architecture
nav_order: 10
---

# Data Model

```mermaid
  erDiagram
      Pipeline {
      }

      Branch {
      }

      BranchLink {
      }

      Step {
      }

      Transition {
      }

      Advancement {
      }

      Job {
      }

      Execution {
      }

      Run {
      }

      Availability {
      }

      Result {
      }

      Process {
      }

      Tuple {
      }

      Pipeline ||--o{ Branch : "has many"
      Pipeline ||--o{ Step : "has many"
      Pipeline ||--o{ Tuple : "has many"
      Branch ||--o{ Step : "has many"
      Branch ||--o{ Transition : "has many"
      Branch ||--o{ BranchLink : "parent of"
      Branch ||--o{ BranchLink : "child of"
      Step ||--o| Job : "has one"
      Step ||--o| Transition : "in_step"
      Step ||--o| Transition : "out_step"
      Transition ||--o{ Advancement : "has many"
      Job ||--o{ Execution : "has many"
      Execution ||--o| Availability : "has one"
      Execution ||--o| Run : "has one"
      Execution ||--o| Result : "has one"
      Process ||--o{ Advancement : "has many"
      Process ||--o{ Availability : "has many"
```
