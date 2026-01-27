🧭 Day 5 --- Execution Query & Cancellation Semantics
===================================================

Goal
----

The objective of Day 5 was to complete the **end-to-end execution lifecycle** from TaskPilot by enabling:

1.  Execution state query

2.  Execution cancellation

3.  Correct propagation of execution-layer semantics

At the end of this day, TaskPilot provides full visibility and control over task execution without directly managing execution logic itself.

* * * * *

Architectural Context
---------------------

TaskPilot continues to operate strictly as a **control plane**.

It does not:

-   execute tasks

-   manage retries

-   determine execution state transitions

Instead, it delegates execution responsibility entirely to the **Distributed Task Platform (DTP)**.

`User
  ↓
TaskPilot (control plane)
  ↓
Distributed Task Platform (execution plane)`

TaskPilot's role is to:

-   orchestrate intent

-   route commands

-   surface execution results transparently

* * * * *

New APIs Introduced
-------------------

### Execution Query

`GET /api/tasks/{id}/execution`

Behavior:

-   Load WebTask

-   Verify scheduler binding exists

-   Delegate to scheduler:

    `GET /tasks/{schedulerTaskId}`

-   Return execution-layer response directly

This endpoint acts as a **read-through proxy** to the execution engine.

* * * * *

### Execution Cancel

`PUT /api/tasks/{id}/execution/cancel`

Behavior:

-   Load WebTask

-   Validate scheduler binding

-   Forward cancel intent to scheduler:

    `PUT /tasks/{schedulerTaskId}/cancel`

TaskPilot does not attempt to interpret execution logic.

It simply forwards user intent to the engine that owns execution state.

* * * * *

HTTP Semantics Design
---------------------

A key design decision on Day 5 was **how TaskPilot handles execution-layer errors**.

### Initial issue

When attempting to cancel a task that had already completed, the scheduler returned:

`409 Conflict
Cannot cancel task in state SUCCEEDED`

Initially, TaskPilot surfaced this as `500 Internal Server Error`, which was misleading.

This violated service boundary principles.

* * * * *

Corrected Behavior
------------------

TaskPilot was updated to:

-   preserve the scheduler's HTTP status

-   pass through execution-layer error messages

-   avoid masking execution semantics

Implementation used `RestClient.exchange(...)` instead of `.retrieve()`.

As a result:

| Scenario | Result |
| --- | --- |
| Cancel running task | 200 / 204 |
| Cancel completed task | 409 Conflict |
| Cancel unbound task | 409 Conflict (control-plane validation) |

This ensures that **execution ownership remains with the scheduler**.

* * * * *

Example Execution Flow
----------------------

### 1\. Create WebTask

`POST /api/tasks`

`{
  "id": 5,
  "schedulerTaskId": null
}`

* * * * *

### 2\. Bind Scheduler

`POST /api/tasks/5/scheduler`

`{
  "schedulerTaskId": 2
}`

* * * * *

### 3\. Query Execution

`GET /api/tasks/5/execution`

Response forwarded directly from scheduler:

`{
  "id": 2,
  "status": "SUCCEEDED",
  "attemptCount": 1,
  "retryState": "COMPLETED"
}`

* * * * *

### 4\. Cancel Execution

`PUT /api/tasks/5/execution/cancel`

Response:

`HTTP/1.1 409 Conflict
{"message":"Cannot cancel task in state SUCCEEDED"}`

This accurately reflects execution-layer reality.

* * * * *

Important Observation
---------------------

Although the WebTask remains in `PENDING` state, the execution-layer task may already be terminal.

This distinction is intentional.

-   WebTask represents **control-plane intent**

-   SchedulerTask represents **execution-plane reality**

Synchronization between these layers is deferred to future iterations.

This mirrors real-world systems where:

-   control state and execution state are not always immediately consistent

-   execution systems remain the source of truth for runtime status

* * * * *

Key Engineering Takeaways
-------------------------

### 1\. Execution ownership must not leak upward

TaskPilot never attempts to:

-   reclassify execution states

-   reinterpret retry logic

-   override scheduler decisions

This avoids tight coupling between services.

* * * * *

### 2\. Error propagation matters

Returning `500` for valid execution-layer rejections hides system meaning.

Correctly propagating `409 Conflict` makes the system:

-   debuggable

-   predictable

-   production-realistic

* * * * *

### 3\. Control plane ≠ execution plane

Day 5 reinforced a fundamental distributed systems principle:

> The service that owns state must own its semantics.

TaskPilot orchestrates --- it does not execute.

* * * * *

Current System State
--------------------

At the end of Day 5:

✅ Full end-to-end execution loop implemented\
✅ Scheduler integration stable\
✅ Execution query and cancel exposed cleanly\
✅ Correct HTTP semantics across service boundaries
