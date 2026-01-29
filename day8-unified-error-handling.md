Day 8 --- Execution Cancel & Unified Error Handling
=================================================

Goal
----

Finalize execution-level cancellation behavior and improve API consistency by unifying error handling between TaskPilot and the Scheduler system.

By the end of today, the system should:

-   Support cancelling execution via `/execution/cancel`

-   Clearly distinguish **web task state** vs **scheduler execution state**

-   Return consistent API error responses for TaskPilot-originated failures

-   Preserve scheduler responses without over-engineering

* * * * *

What I Worked On
----------------

### 1\. Clarified Cancellation Semantics

At this stage, TaskPilot exposes **two different cancel concepts**:

| Endpoint | Responsibility |
| --- | --- |
| `/api/tasks/{id}/cancel` | Cancels the *web task record* |
| `/api/tasks/{id}/execution/cancel` | Cancels the *scheduler execution* |

This separation is intentional and mirrors real production systems:

-   **Web task** = business-level object

-   **Scheduler task** = execution-level object

Only tasks that have been bound to the scheduler can be execution-cancelled.

* * * * *

### 2\. Improved `cancelExecution()` Behavior

Previously, when a task was not yet bound to the scheduler, the service returned:

`ResponseEntity.status(409).body("Task is not bound to scheduler yet.")`

This caused inconsistent response formats across APIs.

#### Updated logic

`public ResponseEntity<String> cancelExecution(Long webTaskId) {
    TaskEntity task = getOrThrow(webTaskId);

    Long schedulerId = task.getSchedulerTaskId();
    if (schedulerId == null) {
        throw new ConflictException("Task is not bound to scheduler yet.");
    }

    return schedulerClient.cancelTask(schedulerId);
}`

Now:

-   TaskPilot-owned validation errors throw domain exceptions

-   Scheduler responses are passed through transparently

* * * * *

### 3\. Unified Error Handling via GlobalExceptionHandler

TaskPilot now handles its own domain errors consistently using centralized exception handling.

Example:

`@ExceptionHandler(ConflictException.class)
public ResponseEntity<ApiError> handleConflict(
        ConflictException ex,
        HttpServletRequest req
) {
    ApiError err = new ApiError(
            409,
            "Conflict",
            ex.getMessage(),
            req.getRequestURI()
    );
    return ResponseEntity.status(409).body(err);
}`

This ensures:

-   Clean JSON error responses

-   Consistent HTTP semantics

-   No duplicated error logic in controllers

* * * * *

### 4\. Clear Responsibility Boundary

A key design decision today:

> TaskPilot is responsible for **business rules**,\
> Scheduler is responsible for **execution mechanics**.

Therefore:

-   TaskPilot validates:

    -   Task existence

    -   Scheduler binding state

-   Scheduler controls:

    -   Execution lifecycle

    -   Cancellation success/failure

    -   Retry behavior

TaskPilot does **not** reinterpret scheduler state --- it only surfaces it.

This keeps integration clean and realistic.

* * * * *

Example Behavior
----------------

### Case 1 --- Task not bound to scheduler

`PUT /api/tasks/7/execution/cancel`

Response:

`{
  "status": 409,
  "error": "Conflict",
  "message": "Task is not bound to scheduler yet.",
  "path": "/api/tasks/7/execution/cancel"
}`

Handled fully inside TaskPilot.

* * * * *

### Case 2 --- Task already completed in scheduler

Scheduler responds:

`{"message":"Cannot cancel task in state SUCCEEDED"}`

TaskPilot passes this through unchanged.

This reflects real-world distributed systems behavior.

* * * * *

Key Takeaways
-------------

-   Execution cancellation is **not the same** as task cancellation

-   Scheduler integration requires strict responsibility boundaries

-   Centralized exception handling dramatically improves API quality

-   Avoid over-wrapping downstream system responses prematurely

* * * * *

Status After Day 8
------------------

✅ Execution cancel endpoint implemented\
✅ Domain errors unified via GlobalExceptionHandler\
✅ Scheduler integration preserved cleanly\
✅ API semantics now production-grade
