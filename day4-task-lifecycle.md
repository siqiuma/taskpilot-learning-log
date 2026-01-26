🛠️ TaskPilot --- Day 4
=====================

**Retries, Backoff Scheduling, and Rerunnable Workflows**

Day 4 focused on making TaskPilot execution-aware by introducing retry behavior, scheduling controls, and state transitions that support rerunnable workflows.

With persistence and cancellation in place, the system now models how tasks behave under real execution conditions: tasks can start, succeed, fail, and retry with a controlled backoff policy.

* * * * *

🎯 Objective
------------

The goals for Day 4 were to:

-   implement task execution lifecycle transitions

-   introduce retry logic with attempt tracking

-   schedule retries using `nextRunAt` (backoff)

-   ensure failed tasks become rerunnable safely

-   enforce domain invariants for all execution-related operations

-   expose worker-oriented APIs to simulate task processing

* * * * *

✅ Completed Work
----------------

### 1\. Execution State Transitions

Domain-level lifecycle operations were added to support execution:

-   `start(workerId)`

    -   transitions `PENDING → RUNNING`

    -   records `workerId` and `processingStartedAt`

-   `succeed()`

    -   transitions `RUNNING → SUCCEEDED`

    -   records `completedAt`

    -   clears retry-related metadata (`nextRunAt`, `lastError`, `retryState`)

These transitions are validated and enforced in the domain layer.

* * * * *

### 2\. Retry Logic with Attempt Tracking

A failure workflow was added:

-   `failAndScheduleRetry(errorMessage)`

Behavior:

-   increments `attemptCount`

-   records failure details (`lastError`)

-   if attempts remain:

    -   transitions back to `PENDING`

    -   sets `retryState = RETRY_SCHEDULED`

    -   schedules `nextRunAt` using a backoff delay

    -   clears execution ownership fields (`workerId`, `processingStartedAt`) to make the task safe to rerun

-   if attempts are exhausted:

    -   transitions to terminal `FAILED`

    -   sets `retryState = EXHAUSTED`

    -   records `completedAt`

This provides controlled retries while preventing infinite retry loops.

* * * * *

### 3\. Backoff Scheduling Enforcement (`nextRunAt`)

To prevent early retries, task start behavior enforces scheduling:

-   if `nextRunAt` is in the future, starting the task is rejected

Invalid early execution attempts return:

-   `HTTP 409 Conflict`

This ensures retries follow backoff timing and makes task execution predictable under repeated failures.

* * * * *

### 4\. Worker-Oriented REST APIs

To simulate worker behavior end-to-end, new endpoints were introduced:

`PUT /api/tasks/{id}/start
PUT /api/tasks/{id}/succeed
PUT /api/tasks/{id}/fail`

-   `start` accepts `workerId`

-   `fail` accepts an `errorMessage`

-   transitions and retry scheduling are controlled entirely by domain rules

* * * * *

### 5\. End-to-End Validation

The following behaviors were verified through API calls:

-   starting a `PENDING` task transitions it to `RUNNING`

-   failing a `RUNNING` task increments `attemptCount`

-   when attempts remain, failure schedules retry and returns task to `PENDING`

-   `nextRunAt` prevents early restart and returns `409` until eligible

-   once max attempts are reached, the task becomes terminal `FAILED`

This confirmed correct behavior across controller → service → domain → database.

* * * * *

✅ Current System Capabilities
-----------------------------

At the end of Day 4, TaskPilot now supports:

-   execution-aware task lifecycle modeling

-   rerunnable workflows (returning tasks to `PENDING` after failure)

-   retry limits and attempt tracking

-   backoff scheduling via `nextRunAt`

-   worker assignment tracking (`workerId`, processing timestamps)

-   domain-enforced invariants with consistent HTTP error semantics

This significantly increases realism and production alignment.

* * * * *

🧠 Key Takeaways
----------------

-   Retries should be modeled explicitly, not handled implicitly by client-side loops.

-   Backoff scheduling (`nextRunAt`) prevents hot-loop retries and improves system stability.

-   Clearing execution ownership fields on retry prevents "stuck" tasks and enables safe reruns.

-   Domain-driven transitions provide correctness guarantees across all entry points.

-   A system becomes robust when it can safely handle failure, not just success.

* * * * *

⏭️ Next Steps
-------------

Day 5 will focus on queue behavior and claiming logic:

-   querying runnable tasks (`PENDING` with `nextRunAt <= now`)

-   safely claiming tasks using concurrency-safe patterns

-   preventing double-processing under concurrent workers

-   introducing repository queries and transactional boundaries

This will move TaskPilot closer to a true orchestration control plane.
