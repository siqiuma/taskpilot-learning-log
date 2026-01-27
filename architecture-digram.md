## High-Level Architecture (Control Plane vs Execution Plane)

                        ┌──────────────────────────┐
                        │          User            │
                        │  (curl / UI / API call)  │
                        └─────────────┬────────────┘
                                      │
                                      ▼
                    ┌────────────────────────────────┐
                    │          TaskPilot              │
                    │        (Control Plane)          │
                    │                                  │
                    │  - Create WebTask                │
                    │  - Bind SchedulerTask            │
                    │  - Expose execution endpoints    │
                    │                                  │
                    │  /api/tasks                      │
                    │  /api/tasks/{id}/scheduler       │
                    │  /api/tasks/{id}/execution       │
                    │  /api/tasks/{id}/execution/cancel│
                    └─────────────┬──────────────────┘
                                  │
                   HTTP (REST API)│
                                  ▼
          ┌────────────────────────────────────────────┐
          │      Distributed Task Platform (DTP)       │
          │           (Execution Plane)                │
          │                                            │
          │  - Task queue                              │
          │  - Worker loop                             │
          │  - Retry logic                             │
          │  - State transitions                       │
          │                                            │
          │  /tasks                                    │
          │  /tasks/{id}                               │
          │  /tasks/{id}/cancel                        │
          └─────────────┬──────────────────────────────┘
                        │
                        ▼
                 ┌──────────────────┐
                 │   PostgreSQL     │
                 │                  │
                 │  webdb  (control)│
                 │  dtp    (execute)│
                 └──────────────────┘
## Execution Lifecycle (Create → Bind → Auto-Execute)
```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant TP as TaskPilot (Control Plane)
    participant DTP as DTP Scheduler (Execution Plane)
    participant DBW as Postgres webdb
    participant DBD as Postgres dtp

    U->>TP: POST /api/tasks
    TP->>DBW: INSERT WebTask (PENDING)
    DBW-->>TP: webTaskId

    U->>TP: POST /api/tasks/{id}/scheduler
    TP->>DTP: POST /tasks (payload JSON)
    DTP->>DBD: INSERT SchedulerTask
    DBD-->>DTP: schedulerTaskId
    DTP-->>TP: schedulerTaskId

    TP->>DBW: UPDATE WebTask.scheduler_task_id
    TP-->>U: 200 OK

    Note over DTP,DBD: Worker loop executes task automatically
    DTP->>DBD: RUNNING → SUCCEEDED

```


## Execution Query
```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant TP as TaskPilot
    participant DTP as Scheduler
    participant DBW as Postgres webdb

    U->>TP: GET /api/tasks/{id}/execution
    TP->>DBW: load scheduler_task_id
    DBW-->>TP: schedulerTaskId

    alt scheduler_task_id is null
        TP-->>U: 409 Task not bound
    else bound
        TP->>DTP: GET /tasks/{schedulerTaskId}
        DTP-->>TP: execution JSON
        TP-->>U: 200 OK (pass-through)
    end
```

## Cancel flow
```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant TP as TaskPilot
    participant DTP as Scheduler
    participant DBW as Postgres webdb

    U->>TP: PUT /api/tasks/{id}/execution/cancel
    TP->>DBW: load scheduler_task_id
    DBW-->>TP: schedulerTaskId

    alt not bound
        TP-->>U: 409 Conflict
    else bound
        TP->>DTP: PUT /tasks/{schedulerTaskId}/cancel

        alt cancellable
            DTP-->>TP: 200 OK
            TP-->>U: 200 OK
        else terminal state
            DTP-->>TP: 409 Conflict
            TP-->>U: 409 Conflict
        end
    end
```

### ✅ Separation of concerns

-   Control plane = intent

-   Execution plane = runtime

### ✅ Ownership boundaries

-   TaskPilot owns orchestration

-   Scheduler owns execution semantics

### ✅ Correct error propagation

-   Scheduler decides what is cancellable

-   TaskPilot does not override runtime truth

### ✅ Real-world parallels

This architecture directly maps to:

| Your Project | Real System |
| --- | --- |
| TaskPilot | Kubernetes API Server |
| DTP | kubelet / controller |
| WebTask | Pod spec |
| SchedulerTask | Pod runtime |
| /execution | kubectl get pod |

TaskPilot acts as a control plane that manages task intent and orchestration, while execution is fully delegated to a distributed scheduler. Execution endpoints exist to observe and control runtime behavior without coupling the control plane to execution semantics.
