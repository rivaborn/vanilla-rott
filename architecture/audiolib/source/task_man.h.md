# audiolib/source/task_man.h

## File Purpose
Public header for a low-level timer task scheduler. Defines the interface for registering, managing, and dispatching time-based tasks with configurable execution rates and priorities. Designed for use in both regular code and interrupt contexts.

## Core Responsibilities
- Define task structure and error codes for task management
- Provide scheduling/termination API for time-based tasks
- Manage task dispatch and rate control
- Track interrupt execution context for code that operates in both environments
- Provide memory locking primitives for interrupt-safe critical sections

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `TASK_ERRORS` | enum | Error/status codes: `TASK_Ok` (0), `TASK_Error` (-1), `TASK_Warning` (-2) |
| `task` | struct | Represents a scheduled task with linked-list pointers, callback, data, rate, priority, and active state |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TS_InInterrupt` | volatile int | global | Flag indicating whether code is executing within a taskman interrupt handler |

## Key Functions / Methods

### TS_ScheduleTask
- Signature: `task *TS_ScheduleTask( void (*Function)(task *), int rate, int priority, void *data )`
- Purpose: Register a new task for periodic execution
- Inputs: Function pointer (task service routine), rate (execution period), priority level, opaque data pointer
- Outputs/Return: Pointer to created `task` structure (for later termination/modification)
- Side effects: Allocates task structure, modifies interrupt-driven task queue
- Calls: Not inferable from this file
- Notes: Rate and priority determine execution order and frequency

### TS_Dispatch
- Signature: `void TS_Dispatch( void )`
- Purpose: Process all scheduled tasks whose execution criteria are met
- Inputs: None
- Outputs/Return: None
- Side effects: Invokes task callbacks; modifies task counters
- Calls: Indirectly calls each registered task's `TaskService` callback
- Notes: Likely called from timer interrupt or main loop

### TS_Terminate
- Signature: `int TS_Terminate( task *ptr )`
- Purpose: Unregister and remove a scheduled task
- Inputs: Pointer to task structure to remove
- Outputs/Return: Status code (`TASK_Ok`, `TASK_Error`, etc.)
- Side effects: Deallocates task; modifies task queue
- Calls: Not inferable from this file

### TS_SetTaskRate
- Signature: `void TS_SetTaskRate( task *Task, int rate )`
- Purpose: Dynamically adjust execution rate of an existing task
- Inputs: Task pointer, new rate value
- Outputs/Return: None
- Side effects: Modifies task's rate field
- Calls: Not inferable from this file

Other functions (`TS_Shutdown`, `TS_LockMemory`, `TS_UnlockMemory`) are trivial wrappers; see notes below.

## Control Flow Notes
This module implements a cooperative task scheduler likely integrated with hardware timer interrupts. Tasks register with a callback, rate, and priority; `TS_Dispatch()` is invoked periodically (from interrupt or main loop) to execute ready tasks. The `TS_InInterrupt` flag allows shared code to detect execution context and adjust synchronization accordingly. Memory locking functions protect critical sections during interrupt-unsafe operations.

## External Dependencies
- **No includes visible** in this header (pure interface definition)
- Clients must include this header to use the task scheduling API
- Task service callbacks receive task structure pointer for access to registered data
