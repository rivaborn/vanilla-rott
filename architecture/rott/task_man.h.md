# rott/task_man.h

## File Purpose
Public header for a low-level timer task scheduler that manages periodic task execution with priority support. Provides interrupt-safe scheduling and dispatching of callback-based tasks at specified rates.

## Core Responsibilities
- Define task structure and scheduler interface
- Register and manage periodic tasks with rate and priority
- Dispatch scheduled tasks based on elapsed time
- Track interrupt context to prevent unsafe concurrent access
- Provide memory locking for interrupt-safe operation
- Handle task lifecycle (schedule, terminate, rate adjustment)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `task` | struct | Linked-list node representing a scheduled task with callback, data, rate, and execution state |
| `TASK_ERRORS` | enum | Error return codes (Warning, Error, Ok) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TS_InInterrupt` | volatile int | global | Flag indicating execution context; TRUE during timer interrupt, FALSE in main loop |

## Key Functions / Methods

### TS_ScheduleTask
- **Signature:** `task *TS_ScheduleTask( void ( *Function )( task * ), int rate, int priority, void *data )`
- **Purpose:** Register a new periodic task to be executed by the scheduler
- **Inputs:** Function pointer (callback), execution rate (timer units), priority level, opaque data pointer
- **Outputs/Return:** Pointer to allocated task structure (used for later manipulation), or NULL on failure
- **Side effects:** Allocates memory, inserts task into scheduler's linked list, modifies global task state
- **Calls:** Not inferable from header
- **Notes:** Task callback receives task pointer as argument; rate controls execution frequency; priority determines execution order

### TS_Dispatch
- **Signature:** `void TS_Dispatch( void )`
- **Purpose:** Execute all scheduled tasks whose timer counters have expired
- **Inputs:** None (operates on global task queue)
- **Outputs/Return:** None
- **Side effects:** Invokes task callbacks, modifies task `count` fields, may set `TS_InInterrupt` flag
- **Calls:** Calls registered task callbacks indirectly
- **Notes:** Likely called from timer interrupt handler; should be fast

### TS_Terminate
- **Signature:** `int TS_Terminate( task *ptr )`
- **Purpose:** Stop and remove a scheduled task
- **Inputs:** Task pointer from TS_ScheduleTask
- **Outputs/Return:** Error code (TASK_Ok, TASK_Error, or TASK_Warning)
- **Side effects:** Removes task from linked list, frees memory, invalidates task pointer
- **Calls:** Not inferable

### TS_SetTaskRate
- **Signature:** `void TS_SetTaskRate( task *Task, int rate )`
- **Purpose:** Modify the execution rate of an already-scheduled task
- **Inputs:** Task pointer, new rate value
- **Outputs/Return:** None
- **Side effects:** Updates task rate; next execution timing may change
- **Calls:** Not inferable

### TS_LockMemory / TS_UnlockMemory
- **Signature:** `int TS_LockMemory( void )` / `void TS_UnlockMemory( void )`
- **Purpose:** Protect memory region from interrupt-handler access (for near/far pointer safety on DOS/protected mode)
- **Inputs/Return:** TS_LockMemory returns error code; TS_UnlockMemory returns void
- **Side effects:** Changes interrupt state or memory protection; must be paired
- **Notes:** Likely DOS-era memory model workaround; critical for async-safe task manipulation

### TS_Shutdown
- **Signature:** `void TS_Shutdown( void )`
- **Purpose:** Cleanup and terminate the task scheduler
- **Inputs/Return:** None
- **Side effects:** Deallocates all tasks, stops interrupt handling, clears global state
- **Notes:** Should be called at engine shutdown

## Control Flow Notes
This scheduler operates as an interrupt-driven system: a timer interrupt periodically calls `TS_Dispatch()`, which executes ready tasks by invoking their callbacks. The `TS_InInterrupt` flag allows tasks to detect execution context (interrupt vs. main loop) for safe data access. Main code schedules tasks via `TS_ScheduleTask()` and can adjust rates dynamically.

## External Dependencies
- None visible; self-contained interface with no included headers shown
