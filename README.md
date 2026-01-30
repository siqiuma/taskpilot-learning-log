TaskPilot responsibilities (source of truth = Web DB)
-----------------------------------------------------

-   **Owns WebTask lifecycle metadata**: create/read/cancel (your local state like `PENDING/CANCELED` if you have it).

-   **Creates the linkage** between a WebTask and a Scheduler task (`schedulerTaskId`) via **bind**.

-   **Provides an execution "view"** by querying the scheduler and projecting the scheduler response into a stable `ExecutionView`.

-   **Forwards execution commands** (e.g., cancel) to the scheduler, and maps scheduler outcomes into meaningful HTTP responses.

Explicit non-responsibilities (source of truth = Scheduler DB)
--------------------------------------------------------------

-   **Does not run tasks** (no worker loop, no processing).

-   **Does not decide execution status** (`SUCCEEDED/FAILED/RUNNING/CANCELED`) --- it only *reports* what the scheduler says.

-   **Does not retry/backoff/claim tasks** --- all execution semantics live in the scheduler.

-   **Does not guarantee real-time state**; execution state is **eventually consistent** because it's pulled from a separate system.

### One sentence

> "TaskPilot owns the user-facing task record and the cross-system link, while the scheduler is the source of truth for execution state; TaskPilot only queries/forwards and never mutates execution status."
