🧭 Day 4 --- TaskPilot ↔ Distributed Task Platform Integration
============================================================

Goal
----

The goal of Day 4 was to complete the **first real integration** between:

-   **TaskPilot** --- Web control plane

-   **Distributed Task Platform (DTP)** --- Execution engine

This marks the transition from isolated services to a **true multi-service system**.

At the end of Day 4, a task created in TaskPilot can be:

1.  Bound to the scheduler

2.  Executed asynchronously by DTP

3.  Tracked through a shared execution identifier

* * * * *

Architecture Context
--------------------

TaskPilot is intentionally designed as a **control-plane service**.

It does **not** execute tasks itself.\
Instead, it orchestrates execution by delegating work to a separate scheduler service.

`TaskPilot (Web API)  ─────▶  Distributed Task Platform
        |                              |
        |                              |
     WebTask                     SchedulerTask`

This separation mirrors real-world systems such as:

-   Airflow (UI vs executor)

-   Temporal (workflow vs worker)

-   Kubernetes (API server vs kubelet)

* * * * *

Integration Design
------------------

### Responsibilities

| Component | Responsibility |
| --- | --- |
| TaskPilot | Task creation, orchestration, ownership |
| DTP | Task execution, retries, state transitions |

### Contract Between Services

TaskPilot communicates with DTP using HTTP APIs:

| Operation | Method | Endpoint |
| --- | --- | --- |
| Create scheduler task | POST | `/tasks` |
| Query execution | GET | `/tasks/{id}` |
| Cancel execution | PUT | `/tasks/{id}/cancel` |

DTP returns a unique `schedulerTaskId`, which TaskPilot stores for future tracking.

* * * * *

Database Change
---------------

A new column was added to TaskPilot's task table:

`scheduler_task_id BIGINT`

This allows TaskPilot to maintain a stable reference to the execution-layer task.

Flyway migration:

`V2__add_scheduler_task_id.sql`

This preserves clean ownership boundaries:

-   TaskPilot owns WebTask lifecycle

-   DTP owns execution lifecycle

* * * * *

Scheduler Client Implementation
-------------------------------

TaskPilot introduces a dedicated integration component:

`SchedulerClient`

Responsibilities:

-   Encapsulate HTTP communication with DTP

-   Prevent scheduler logic from leaking into controller or service layers

-   Allow future replacement (e.g., async, message-based integration)

### Supported operations

-   `createTask(type, payloadJson)`

-   `getTask(id)`

-   `cancelTask(id)`

This keeps integration logic isolated and testable.

* * * * *

Binding Flow
------------

A new endpoint was added to TaskPilot:

`POST /api/tasks/{id}/scheduler`

### Binding steps:

1.  Load WebTask

2.  Ensure task is not already bound

3.  Construct payload JSON:

`{
  "webTaskId": 4,
  "type": "demo",
  "payload": "hello"
}`

1.  Call scheduler `POST /tasks`

2.  Receive `schedulerTaskId`

3.  Persist scheduler reference in TaskPilot DB

This creates a durable link between control-plane and execution-plane tasks.

* * * * *

End-to-End Verification
-----------------------

### 1\. Create Web Task

`POST http://localhost:8081/api/tasks`

Result:

`{
  "id": 4,
  "schedulerTaskId": null
}`

* * * * *

### 2\. Bind Scheduler Task

`POST http://localhost:8081/api/tasks/4/scheduler`

Result:

`{
  "schedulerTaskId": 1
}`

* * * * *

### 3\. Query Scheduler Directly

`GET http://localhost:8080/tasks/1`

Result:

`{
  "id": 1,
  "status": "SUCCEEDED",
  "payload": "{\"webTaskId\":4,\"type\":\"demo\",\"payload\":\"hello\"}"
}`

This confirms:

✅ TaskPilot successfully delegated execution\
✅ DTP executed task independently\
✅ Task identity linkage works end-to-end

* * * * *

Key Engineering Takeaways
-------------------------

### 1\. Clear Service Boundaries Matter

TaskPilot does not know:

-   how execution works

-   how retries are implemented

-   how workers operate

It only knows **how to orchestrate**.

This clean separation improves:

-   testability

-   scalability

-   future extensibility

* * * * *

### 2\. Control Plane vs Execution Plane Is a Powerful Pattern

This architecture mirrors real production systems:

-   Web/UI service controls intent

-   Execution service handles runtime complexity

It avoids tightly coupling user-facing APIs with execution semantics.

* * * * *

### 3\. Integration Is More Valuable Than Features

The most important outcome of Day 4 was not new code ---\
it was **a working system boundary**.

Two independently running services now cooperate through a defined contract.

This is the foundation for:

-   execution monitoring

-   cancellation

-   retries

-   observability

-   future async messaging

* * * * *

Status
------

✅ Day 4 completed successfully\
✅ First end-to-end integration achieved\
➡️ Ready for Day 5: execution query and cancellation endpoints
