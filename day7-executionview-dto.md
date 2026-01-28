Day 7 --- Execution View DTO & Contract Isolation
===============================================

Objective
---------

Refactor the execution query API to return a **TaskPilot-owned execution contract**, instead of directly proxying raw scheduler JSON responses.

This establishes a clear boundary between:

-   **control plane (TaskPilot)** and

-   **execution plane (Distributed Task Platform)**

and prevents internal scheduler implementation details from leaking to API consumers.

* * * * *

Problem
-------

Initially, the endpoint:

`GET /api/tasks/{id}/execution`

returned a raw `ResponseEntity<String>` directly from the scheduler service.

This caused several issues:

-   TaskPilot's API contract depended on scheduler response structure

-   Any scheduler field changes would break clients

-   No ability to derive or reshape execution data

-   Weak separation of responsibilities between systems

* * * * *

Design Decision
---------------

Introduce a dedicated execution projection owned by TaskPilot.

The system now uses **two DTO layers**:

### 1\. SchedulerTaskDto (integration DTO)

Represents the exact response format returned by the scheduler service.

Used only for:

-   JSON deserialization

-   internal integration logic

Not exposed to API consumers.

### 2\. ExecutionView (API contract)

Represents TaskPilot's stable execution view.

Returned by:

`GET /api/tasks/{id}/execution`

This contract is:

-   owned by TaskPilot

-   decoupled from scheduler internals

-   safe to evolve independently

* * * * *

Implementation
--------------

### Scheduler integration

-   Scheduler responses are deserialized into `SchedulerTaskDto`

-   JSON is no longer passed through as raw strings

`ResponseEntity<SchedulerTaskDto> schedulerRes =
        schedulerClient.getTask(schedulerTaskId);`

* * * * *

### ExecutionView mapping

The service layer maps scheduler data into TaskPilot's execution view:

`ExecutionView view = new ExecutionView();
view.setSchedulerTaskId(s.getId());
view.setStatus(s.getStatus());
view.setAttemptCount(s.getAttemptCount());
view.setMaxAttempts(s.getMaxAttempts());
view.setNextRunAt(s.getNextRunAt());
view.setRetryState(s.getRetryState());
view.setLastError(s.getLastError());
view.setScheduledFor(s.getScheduledFor());
view.setProcessingStartedAt(s.getProcessingStartedAt());
view.setCompletedAt(s.getCompletedAt());
view.setWorkerId(s.getWorkerId());`

This ensures:

-   scheduler fields are translated intentionally

-   API consumers see only curated execution data

* * * * *

API Result
----------

`GET /api/tasks/{id}/execution`

returns:

`{
  "schedulerTaskId": 4,
  "status": "SUCCEEDED",
  "attemptCount": 1,
  "maxAttempts": 3,
  "nextRunAt": null,
  "retryState": "COMPLETED",
  "lastError": null,
  "scheduledFor": "2026-01-27T19:30:24.639878Z",
  "processingStartedAt": "2026-01-27T19:30:24.664782Z",
  "completedAt": "2026-01-27T19:30:24.724168Z",
  "workerId": "5b1a4e42-8be0-4f5b-8696-40f5e946bebb"
}`

The response format is now fully controlled by TaskPilot.

* * * * *

Result
------

-   Scheduler JSON is no longer leaked to clients

-   Execution queries return a stable, intentional API contract

-   TaskPilot cleanly separates orchestration logic from execution mechanics

-   The system now supports future enhancements such as:

    -   execution latency calculations

    -   retries and history views

    -   richer execution analytics

* * * * *

Key Engineering Takeaway
------------------------

> Control-plane services should never expose raw execution-plane responses.\
> Integration DTOs and API contracts must be separated.

This separation enables long-term evolvability and protects downstream consumers from internal system changes.
