🔧 API Usage Examples
---------------------

Below are example `curl` commands demonstrating the full execution lifecycle using TaskPilot.

> Base URL (Web API):\
> `http://localhost:8081`

* * * * *

### 1️⃣ Create Task

Creates a new web task stored in TaskPilot.

`curl -X POST http://localhost:8081/api/tasks\
  -H "Content-Type: application/json"\
  -d '{
    "type": "demo",
    "payload": "hello"
  }'`

**Response (example):**

`{
  "id": 14,
  "status": "PENDING",
  "schedulerTaskId": null
}`

* * * * *

### 2️⃣ Bind Scheduler

Binds the web task to the distributed task scheduler.

`curl -X POST http://localhost:8081/api/tasks/14/scheduler\
  -H "Content-Type: application/json"\
  -d '{
    "schedulerType": "demo-slow"
  }'`

This creates a scheduler-side task and stores `schedulerTaskId` in TaskPilot.

* * * * *

### 3️⃣ Get Execution Status

Retrieves execution state from the scheduler and presents a normalized execution view.

`curl http://localhost:8081/api/tasks/14/execution`

**Response (example):**

`{
  "schedulerTaskId": 9,
  "status": "SUCCEEDED",
  "attemptCount": 1,
  "maxAttempts": 3,
  "retryState": "COMPLETED",
  "scheduledFor": "2026-01-29T20:51:42.077Z",
  "processingStartedAt": "2026-01-29T20:51:42.109Z",
  "completedAt": "2026-01-29T20:51:42.941Z",
  "queueLatencyMs": 32,
  "processingLatencyMs": 64,
  "endToEndLatencyMs": 96
}`

* * * * *

### 4️⃣ Cancel Execution

Cancels a scheduled or running task in the scheduler.

`curl -X PUT http://localhost:8081/api/tasks/14/execution/cancel`

**Response:**

`204 No Content`

> This endpoint is **idempotent** --- repeated cancel requests safely return `204` if the task is already canceled.
