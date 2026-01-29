Day 9 --- Execution Timeline & Derived Latency Metrics
====================================================

Goal
----

Improve execution observability by exposing **derived latency metrics** based on scheduler timestamps.

The objective is to make execution behavior easier to understand by clearly separating:

-   time spent waiting in the queue

-   time spent actively processing

-   total end-to-end execution time

This mirrors how real production systems analyze latency and bottlenecks without requiring a full metrics or monitoring stack.

* * * * *

What I Implemented
------------------

### 1\. Extended `ExecutionView` with Derived Metrics

Added three computed latency fields to the execution response:

-   **`queueLatencyMs`**\
    Time between when a task is scheduled and when processing actually begins.

-   **`processingLatencyMs`**\
    Time spent actively executing the task logic.

-   **`endToEndLatencyMs`**\
    Total time from scheduling to completion.

These values are **derived dynamically** and are not persisted in the database.

* * * * *

### 2\. Latency Computation Logic

Latency values are calculated from scheduler-provided timestamps:

`queueLatencyMs        = processingStartedAt - scheduledFor
processingLatencyMs   = completedAt - processingStartedAt
endToEndLatencyMs     = completedAt - scheduledFor`

Implementation details:

-   Calculations use `Duration.between(...)`

-   If any required timestamp is missing, the corresponding latency is returned as `null`

-   No database schema changes were required

This keeps execution analysis lightweight and deterministic.

* * * * *

### 3\. Clear Responsibility Boundaries

Latency derivation is intentionally handled in **TaskPilot**, not in the scheduler.

Responsibilities remain cleanly separated:

-   **Scheduler**\
    Provides raw execution timestamps

-   **TaskPilot**\
    Interprets execution behavior and exposes meaningful performance insights

This design mirrors real-world systems where application services derive metrics from infrastructure-level execution data.

* * * * *

Example Response
----------------

`{
  "schedulerTaskId": 4,
  "status": "SUCCEEDED",
  "attemptCount": 1,
  "maxAttempts": 3,
  "retryState": "COMPLETED",
  "scheduledFor": "2026-01-27T19:30:24.639878Z",
  "processingStartedAt": "2026-01-27T19:30:24.664782Z",
  "completedAt": "2026-01-27T19:30:24.724168Z",
  "queueLatencyMs": 24,
  "processingLatencyMs": 59,
  "endToEndLatencyMs": 84
}`

From a single response, execution behavior becomes immediately visible:

-   ~24 ms waiting in queue

-   ~59 ms executing business logic

-   ~84 ms total end-to-end latency

* * * * *

Why This Matters
----------------

This enhancement significantly improves the system's **observability and debuggability**:

-   Enables clear reasoning about execution delays

-   Separates queue time from processing time

-   Makes performance tradeoffs easy to explain in interviews

-   Requires no monitoring or metrics infrastructure

This is the type of design detail commonly expected in production backend systems.

* * * * *

Status After Day 9
------------------

✅ Execution timeline fully exposed\
✅ Derived latency metrics computed dynamically\
✅ Clean naming aligned with execution semantics\
✅ No persistence or schema changes required
