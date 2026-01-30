**TaskPilot owns**

-   Create web task

-   Bind to scheduler (store `schedulerTaskId`)

-   Read execution status (normalized view)

-   Cancel execution (request cancellation)

**Scheduler owns**

-   Task state machine (`PENDING/RUNNING/SUCCEEDED/FAILED/CANCELED`)

-   Retry logic, timestamps, worker assignment

-   Final execution outcome

### One sentence

> "TaskPilot owns the user-facing task record and the cross-system link, while the scheduler is the source of truth for execution state; TaskPilot only queries/forwards and never mutates execution status."

-   `POST /api/tasks` → creates web task in TaskPilot DB

-   `POST /api/tasks/{id}/scheduler` → binds to scheduler (idempotent)

-   `GET /api/tasks/{id}/execution` → returns `ExecutionView` (normalized)

-   `PUT /api/tasks/{id}/execution/cancel` → idempotent cancel request (204 on success)

**Failure semantics & status mapping**

TaskPilot is a web layer that *delegates execution to the Scheduler*. TaskPilot does not mutate scheduler state directly (except issuing a cancel request). TaskPilot maps scheduler outcomes into stable API semantics:

-   **409 Conflict**

    -   Task is not bound to a scheduler task (`schedulerTaskId == null`)

    -   Scheduler rejects a cancel due to task state (e.g., already `SUCCEEDED`)

-   **404 Not Found**

    -   Web task not found in TaskPilot DB

    -   Scheduler task missing (cross-system inconsistency: TaskPilot has `schedulerTaskId`, but Scheduler returns 404)

-   **204 No Content (Cancel)**

    -   Cancel request accepted by scheduler (**idempotent**: repeating cancel may still succeed)
**Cancel execution is idempotent.**\
`PUT /api/tasks/{id}/execution/cancel` returns `204 No Content` if the scheduler task is canceled **or already canceled**.\
If the scheduler task is in a terminal non-cancelable state (e.g., `SUCCEEDED`), TaskPilot returns `409 Conflict` with the scheduler's reason.

TaskPilot is a façade over the scheduler for execution operations; it does not mutate execution state itself, and it preserves scheduler failure semantics (409/404) while normalizing success to 204.

### `GET /api/tasks/{id}/execution` --- Execution View (from Scheduler)

TaskPilot returns a normalized execution view by querying the Scheduler task linked via `schedulerTaskId`.

**Semantics**

-   **200 OK**: Scheduler returned 2xx and a task payload was available. TaskPilot maps Scheduler fields into `ExecutionView` (including derived latency fields).

-   **409 Conflict**:

    -   Web task is **not bound** to a scheduler task (`schedulerTaskId == null`)

    -   Scheduler responded **non-2xx** (except 404), or response body was unavailable\
        (TaskPilot treats this as "scheduler task not available")

-   **404 Not Found**:

    -   Web task does not exist in TaskPilot

    -   Scheduler task link exists, but Scheduler returns 404 (cross-system inconsistency)
