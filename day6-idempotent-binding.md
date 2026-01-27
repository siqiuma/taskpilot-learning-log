Day 6 --- Idempotent Scheduler Binding
====================================

Objective
---------

Ensure that binding a Web Task to the distributed task scheduler is **idempotent**, so repeated client requests do not create duplicate scheduler tasks.

In real systems, retries can occur due to:

-   network timeouts

-   client retries

-   accidental double submission

The system must handle these safely.

* * * * *

Problem
-------

Previously, calling:

`POST /api/tasks/{id}/scheduler`

more than once would throw an error or attempt to create a new scheduler task.\
This could lead to **duplicate execution**, which is unacceptable in production systems.

* * * * *

Design Decision
---------------

The binding operation should be **idempotent**:

-   If the WebTask is not yet bound → create a scheduler task

-   If already bound → return the existing binding

This follows standard distributed-system best practices for side-effect operations.

* * * * *

Implementation
--------------

### Service-level idempotency

Updated the scheduler binding logic:

`if (task.getSchedulerTaskId() != null) {
    return task;
}`

This ensures:

-   only the first request creates a scheduler task

-   all subsequent requests safely return the existing association

### Transactional safety

The binding method runs inside a transaction to guarantee consistency between:

-   scheduler task creation

-   persistence of `schedulerTaskId` in the Web DB

### Optimistic locking

Added a `@Version` field to the `TaskEntity` and corresponding database column:

`@Version
private Long version;`

This prevents race conditions where two concurrent requests attempt to bind the same task simultaneously.

* * * * *

Verification
------------

### Step 1 --- Create Web Task

`POST /api/tasks`

Returned:

`{
  "id": 7,
  "schedulerTaskId": null
}`

### Step 2 --- First bind

`POST /api/tasks/7/scheduler`

Returned:

`{
  "schedulerTaskId": 4
}`

### Step 3 --- Second bind (retry)

`POST /api/tasks/7/scheduler`

Returned:

`{
  "schedulerTaskId": 4
}`

No additional scheduler task was created.

The database record remained unchanged, confirming idempotent behavior.

* * * * *

Result
------

-   Scheduler binding is now safe under retries

-   Duplicate task creation is prevented

-   API behaves predictably for clients

This establishes a production-grade guarantee for all side-effect operations.

* * * * *

Key Engineering Takeaway
------------------------

> All APIs that trigger external side effects must be designed as idempotent operations to ensure correctness under retries and partial failures.
