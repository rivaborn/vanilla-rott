# audiolib/public/include/task_man.h

## File Purpose
Public header for a low-level timer-based task scheduler used in the audio library. Manages periodic task execution with priority levels and interrupt-driven dispatch, typical of DOS-era game engine design where timer interrupts drive frame-locked tasks.

## Core Responsibilities
- Define task scheduling primitives (creation, termination, rate adjustment)
- Provide task dispatch mechanism for interrupt handlers or main loop
- Manage task linked lists with priority and execution rates
- Track interrupt context to allow code reuse in interrupt and non-interrupt paths
- Lock/unlock memory for interrupt safety (DOS-specific)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `task` | struct | Task node in doubly-linked list; holds service callback, execution rate, priority, and user data. |
| `TASK_ERRORS` | enum | Standard error return codes (Warning, Error, Ok). |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TS_InInterrupt` | volatile int | global | Flag indicating if code is executing within a timer interrupt; allows conditional logic for ISR-safe operations. |

## Key Functions / Methods

### TS_ScheduleTask
- Signature: `task *TS_ScheduleTask(void (*Function)(task *), int rate, int priority, void *data)`
- Purpose: Create and register a new periodic task into the scheduler.
- Inputs: `Function` (callback), `rate` (execution frequency), `priority` (scheduling order), `data` (task-specific context)
- Outputs/Return: Pointer to allocated task node, or NULL on failure.
- Side effects: Allocates memory, inserts into scheduler's linked list.
- Calls: (not inferable from header)
- Notes: Task callback receives its own task node as argument (self-referential design).

### TS_Dispatch
- Signature: `void TS_Dispatch(void)`
- Purpose: Process all active tasks whose execution counters have elapsed.
- Inputs: None
- Outputs/Return: None
- Side effects: Invokes task callbacks in priority order; modifies task counters.
- Calls: User-provided task callbacks via `TaskService` function pointers.
- Notes: Likely called from timer interrupt or main loop; respects `TS_InInterrupt` state.

### TS_Terminate
- Signature: `int TS_Terminate(task *ptr)`
- Purpose: Remove a task from the scheduler and release its resources.
- Inputs: `ptr` (task node to remove)
- Outputs/Return: Status code (TASK_Ok on success)
- Side effects: Deallocates memory, unlinks task from list.
- Calls: (not inferable from header)

### TS_SetTaskRate
- Signature: `void TS_SetTaskRate(task *Task, int rate)`
- Purpose: Dynamically adjust the execution frequency of an active task.
- Inputs: `Task` (target task), `rate` (new execution interval)
- Outputs/Return: None
- Side effects: Modifies task rate in-place.
- Notes: Can be called from interrupt context.

### TS_Shutdown
- Signature: `void TS_Shutdown(void)`
- Purpose: Cleanly terminate the task scheduler and release all resources.
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates all tasks, resets global state.
- Notes: Likely called on engine shutdown.

### TS_LockMemory / TS_UnlockMemory
- Signature: `int TS_LockMemory(void)` / `void TS_UnlockMemory(void)`
- Purpose: Lock task memory into physical RAM to prevent page faults during timer interrupts (DOS DOS4GW/DPMI specific).
- Inputs: None
- Outputs/Return: Status code (LockMemory only)
- Side effects: Page-lock scheduler and task data structures.
- Notes: DOS-specific safety for interrupt handlers; not typically needed on modern systems.

## Control Flow Notes
This scheduler appears integrated into an **interrupt-driven timing model**: a timer interrupt fires periodically, calls `TS_Dispatch()` (setting `TS_InInterrupt = 1`), which executes eligible tasks based on rate counters and priority. The `volatile count` field in the task struct suggests interrupt-safe decrementing. Tasks can modify themselves or other tasks via their callbacks, provided they respect the interrupt context flag.

## External Dependencies
- None (self-contained public header; implementation in audiolib).
