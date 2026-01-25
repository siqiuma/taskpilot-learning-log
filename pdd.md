📘 Project Design Document
==========================

Task Management Web Application
-------------------------------

*(Integrated with Task Scheduler Service)*

* * * * *

1\. Project Overview
--------------------

### 1.1 Background

Many systems require reliable task execution with clear visibility into task status, execution history, and failure handling. While the Task Scheduler service provides execution and scheduling capabilities, it does not offer a user-facing interface for managing tasks.

This project aims to build a **full-stack Task Management Web Application** that allows users to create, manage, and monitor tasks through a web interface, while delegating execution and scheduling responsibilities to an independent **Task Scheduler service**.

The system follows a **layered architecture** separating product concerns from platform execution logic.

* * * * *

### 1.2 Goals

-   Provide a user-friendly web interface for managing scheduled tasks

-   Enable end-to-end workflows:

    -   Create task → configure schedule → trigger execution → view status

-   Integrate with an existing Task Scheduler service via REST APIs

-   Demonstrate clean system boundaries, service integration, and data consistency handling

-   Deliver a production-like, deployable full-stack system

* * * * *

### 1.3 Non-Goals (Explicitly Out of Scope)

The following are intentionally excluded to keep scope controlled:

-   Real-time execution via WebSocket

-   Distributed worker execution

-   Complex role-based access control (RBAC)

-   Multi-tenant organization support

-   High availability / horizontal scaling

-   UI animation or advanced frontend design

-   AI-driven task logic

* * * * *

2\. System Architecture
-----------------------

### 2.1 High-Level Architecture

`[ React Frontend ]
        |
        | REST APIs
        v
[ Task Management Web Backend ]
        |
        | Internal REST APIs
        v
[ Task Scheduler Service ]`

* * * * *

### 2.2 Architectural Principles

-   **Separation of concerns**

    -   Web App handles user interaction and business context

    -   Scheduler handles execution logic and state machine

-   **Loose coupling**

    -   Integration via REST APIs

    -   Services maintain independent data models

-   **Explicit data contracts**

    -   DTO-based communication between services

-   **Failure-aware design**

    -   Scheduler failures must not corrupt Web App state

* * * * *

3\. Components
--------------

* * * * *

### 3.1 Frontend (React)

Responsibilities:

-   User authentication

-   Task CRUD UI

-   Schedule configuration

-   Execution triggering

-   Status visualization

Key Pages:

-   Login / Register

-   Task List

-   Create / Edit Task

-   Task Detail (execution status & history)

* * * * *

### 3.2 Web Backend (Spring Boot)

Responsibilities:

-   User management and authentication

-   Task metadata persistence

-   API orchestration between frontend and scheduler

-   Aggregation of execution status

-   Validation and error handling

This backend acts as the **application layer**.

* * * * *

### 3.3 Task Scheduler Service (Existing)

Responsibilities:

-   Task state machine

-   Schedule creation

-   Execution triggering

-   Retry and rerunnable workflows

-   Idempotent execution guarantees

This service acts as the **platform/engine layer**.

* * * * *

4\. Data Model (Web Backend)
----------------------------

### 4.1 User

| Field | Type | Description |
| --- | --- | --- |
| id | UUID | Primary key |
| email | String | Unique |
| password_hash | String | Encrypted |
| created_at | Timestamp |  |

* * * * *

### 4.2 Task

| Field | Type | Description |
| --- | --- | --- |
| id | UUID | Primary key |
| user_id | UUID | Owner |
| title | String | Task name |
| description | String | Optional |
| status | Enum | ACTIVE / DISABLED |
| scheduler_task_id | UUID | Reference to scheduler |
| created_at | Timestamp |  |
| updated_at | Timestamp |  |

* * * * *

### 4.3 ScheduleConfig

| Field | Type | Description |
| --- | --- | --- |
| id | UUID | Primary key |
| task_id | UUID | FK |
| schedule_type | Enum | ONE_TIME / INTERVAL |
| schedule_value | String | e.g. cron or interval |
| enabled | Boolean |  |

* * * * *

*(Scheduler service maintains its own internal tables.)*

* * * * *

5\. Core Features
-----------------

* * * * *

### 5.1 Authentication

-   User registration

-   User login

-   JWT-based authentication

-   Authorization enforced on all task APIs

* * * * *

### 5.2 Task Management

Users can:

-   Create tasks

-   Update task metadata

-   Delete tasks (soft delete acceptable)

-   View task list

-   Filter tasks by status

Validation:

-   Title required

-   User can only access own tasks

* * * * *

### 5.3 Scheduler Integration

#### Create Schedule

-   Web backend calls Scheduler API to create a scheduled task

-   Scheduler returns `schedulerTaskId`

-   Web backend persists reference

#### Trigger Execution

-   User triggers execution from UI

-   Web backend forwards request to Scheduler

#### Query Status

-   Web backend queries Scheduler status API

-   Aggregates response for frontend display

* * * * *

### 5.4 Error Handling

Standard error response:

`{
  "errorCode": "TASK_NOT_FOUND",
  "message": "Task does not exist"
}`

Handled cases:

-   Invalid state transitions

-   Duplicate execution requests

-   Scheduler unavailable

-   Validation failures

* * * * *

### 5.5 Idempotency Strategy

-   Web backend enforces idempotency for:

    -   Schedule creation

    -   Execution trigger

-   Duplicate requests return existing schedulerTaskId

-   Prevents inconsistent state during retries

* * * * *

6\. API Design (Web Backend)
----------------------------

### Authentication

-   POST /api/auth/register

-   POST /api/auth/login

### Tasks

-   POST /api/tasks

-   PUT /api/tasks/{id}

-   DELETE /api/tasks/{id}

-   GET /api/tasks

-   GET /api/tasks/{id}

### Scheduling

-   POST /api/tasks/{id}/schedule

-   POST /api/tasks/{id}/trigger

-   GET /api/tasks/{id}/status

* * * * *

7\. Testing Strategy
--------------------

-   Unit tests:

    -   Service layer

    -   State validation logic

-   Integration tests:

    -   Controller endpoints

    -   Scheduler API mocking

-   Goal: coverage of core business logic

* * * * *

8\. Deployment
--------------

-   Dockerized services:

    -   Web backend

    -   Scheduler

    -   PostgreSQL

-   Docker Compose for local environment

-   Optional cloud deployment (AWS / Render / Railway)

* * * * *

9\. Success Criteria
--------------------

The project is considered complete when:

-   User can manage tasks via UI

-   Scheduler integration works end-to-end

-   Task execution status visible

-   System survives repeated requests safely

-   Project can be demonstrated live

-   Design decisions can be explained clearly in interviews

* * * * *

10\. Key Engineering Takeaways 
-----------------------------------------------

-   Separation of platform vs application logic

-   Service integration via REST contracts

-   Idempotent workflows

-   Failure-aware system design

-   Clean API design and validation

-   Ownership of end-to-end delivery
