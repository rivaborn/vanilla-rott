# audiolib/source/task_man.c

## File Purpose

Low-level timer task scheduler for DOS x86 systems. Manages periodic task execution via hardware interrupt (INT 8, the 8253 timer). Provides real-time task scheduling with priority ordering, interrupt-safe operation, and optional memory locking for deterministic behavior.

## Core Responsibilities

- Schedule and manage periodic tasks executed from timer interrupt context
- Control the 8253 timer hardware, calculating interrupt rates from task frequencies
- Maintain doubly-linked task list with priority-based insertion and removal
- Provide two interrupt handling modes: disabled-interrupt and re-entrant with interrupt enabled
- Optionally allocate and switch to dedicated interrupt stack (USESTACK mode)
- Lock memory regions to prevent page faults during interrupt handling (LOCKMEMORY mode)
- Activate/deactivate tasks and adjust their execution rates dynamically

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `task` | struct | Task descriptor: callback, rate, priority, counter, user data, linked-list pointers (defined in task_man.h) |
| `tasklist` | struct | Simple wrapper with `next`/`end` task pointers (unused in practice) |
| `REGS`/`dpmi_regs` | union/struct | CPU register state for DPMI and DOS interrupt calls |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `HeadTask` | task | static | Sentinel node for circular doubly-linked task list |
| `TaskList` | task* | static | Points to HeadTask; all active tasks linked from here |
| `OldInt8` | void (\__interrupt \__far*)() | static | Saved original INT 8 vector for chaining on shutdown |
| `TaskServiceRate` | volatile long | static | Current 8253 timer reload value (higher = slower interrupt rate) |
| `TaskServiceCount` | volatile long | static | Accumulated interrupt count; rolls over and chains to previous INT 8 at 0x10000 |
| `TS_TimesInInterrupt` | volatile int | static | Re-entrancy guard; counts queued interrupts (only when NOINTS not defined) |
| `TS_Installed` | static char | static | Initialization flag; prevents multiple TS_Startup calls |
| `TS_InInterrupt` | volatile int | global | Extern flag indicating current execution is in interrupt context; used by external code |
| `StackSelector` / `StackPointer` | unsigned short / unsigned long | static | DPMI selector and pointer for dedicated interrupt stack (USESTACK only) |
| `oldStackSelector` / `oldStackPointer` | unsigned short / unsigned long | static | Saved original stack during interrupt (USESTACK only) |

## Key Functions / Methods

### TS_ScheduleTask
- **Signature:** `task *TS_ScheduleTask(void (*Function)(task *), int rate, int priority, void *data)`
- **Purpose:** Create and register a new periodic task for execution via timer interrupt
- **Inputs:** Task callback function pointer; rate in Hz; priority level (higher = earlier execution); optional user data pointer
- **Outputs/Return:** Allocated task pointer on success; NULL on allocation failure or TS_Startup error
- **Side effects:** Initializes system via TS_Startup if not yet installed; allocates task memory; modifies TaskList via TS_AddTask; disables/restores interrupts in TS_AddTask
- **Calls:** malloc (or USRHOOKS_GetMem), TS_Startup, TS_SetTimer, TS_AddTask, FreeMem (on error)
- **Notes:** Tasks are created with `active=FALSE`; must call TS_Dispatch to activate all at once. Rate is converted to timer value via TS_SetTimer.

### TS_Shutdown
- **Signature:** `void TS_Shutdown(void)`
- **Purpose:** Disable task scheduler, restore original timer interrupt, and free all resources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Frees all tasks; restores INT 8 vector; resets clock to default; deallocates interrupt stack if USESTACK; unlocks memory if LOCKMEMORY; sets TS_Installed=FALSE
- **Calls:** TS_FreeTaskList, TS_SetClockSpeed, _dos_setvect, deallocateTimerStack, TS_UnlockMemory
- **Notes:** Safe to call even if never TS_ScheduleTask; idempotent via TS_Installed check

### TS_Dispatch
- **Signature:** `void TS_Dispatch(void)`
- **Purpose:** Activate all currently inactive tasks so they run on next timer interrupt
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `active=TRUE` on all tasks in list; disables/restores interrupts during traversal
- **Calls:** DisableInterrupts, RestoreInterrupts, task list traversal
- **Notes:** Typically called once after scheduling all tasks

### TS_Terminate
- **Signature:** `int TS_Terminate(task *NodeToRemove)`
- **Purpose:** Stop and remove a specific task; recalculate optimal timer rate
- **Inputs:** Task pointer to terminate
- **Outputs/Return:** TASK_Ok if found and removed; TASK_Warning if not found in list
- **Side effects:** Removes task from linked list; frees task memory; calls TS_SetTimerToMaxTaskRate; disables/restores interrupts
- **Calls:** DisableInterrupts, RestoreInterrupts, LL_RemoveNode, TS_SetTimerToMaxTaskRate, FreeMem
- **Notes:** Returns TASK_Warning if task not in list (idempotent safety)

### TS_SetTaskRate
- **Signature:** `void TS_SetTaskRate(task *Task, int rate)`
- **Purpose:** Dynamically change the periodic rate of a running task
- **Inputs:** Task pointer; new rate in Hz
- **Outputs/Return:** None
- **Side effects:** Updates task rate via TS_SetTimer (may reprogram 8253); recalculates optimal clock speed; disables/restores interrupts
- **Calls:** DisableInterrupts, RestoreInterrupts, TS_SetTimer, TS_SetTimerToMaxTaskRate
- **Notes:** Task counter continues accumulating; new rate takes effect at next tick

### TS_ServiceSchedule (when NOINTS defined)
- **Signature:** `static void __interrupt __far TS_ServiceSchedule(void)`
- **Purpose:** Interrupt handler for INT 8 (timer tick); executes with interrupts disabled
- **Inputs:** None (x86 interrupt handler)
- **Outputs/Return:** None
- **Side effects:** Updates all active task counters; calls TaskService callbacks when counter >= rate; increments TaskServiceCount; chains to original INT 8 on rollover; issues EOI (0x20 to port 0x20); sets/clears TS_InInterrupt flag
- **Calls:** GetStack/SetStack (if USESTACK), each task's TaskService function pointer, _chain_intr, outp
- **Notes:** Iterates task list and may invoke multiple tasks per interrupt. Uses while loop for tasks with short rate (comment "//JIM" indicates accumulated ticks may trigger task multiple times). Optional dedicated stack switch via USESTACK.

### TS_ServiceScheduleIntEnabled (when NOINTS not defined)
- **Signature:** `static void __interrupt __far TS_ServiceScheduleIntEnabled(void)`
- **Purpose:** Interrupt handler for INT 8; re-entrant design enabling interrupts to allow nested ticks
- **Inputs:** None (x86 interrupt handler)
- **Outputs/Return:** None
- **Side effects:** Increments TS_TimesInInterrupt counter; updates TaskServiceCount and chains on overflow; issues EOI; defers task execution if already in interrupt context; enables/disables interrupts around task loop
- **Calls:** GetStack/SetStack (if USESTACK), task->TaskService for each active task, _enable, _disable, _chain_intr, outp
- **Notes:** Re-entrancy guard: if TS_InInterrupt already set, just increments counter and returns, deferring tasks until outer interrupt finishes. Processes accumulated interrupts in while loop. Different from TS_ServiceSchedule in that interrupts are enabled during task execution.

### TS_SetClockSpeed
- **Signature:** `static void TS_SetClockSpeed(long speed)`
- **Purpose:** Reprogram the 8253 timer with new interrupt rate
- **Inputs:** Timer reload value (0x0001–0xffff; 0x10000 = slowest, default)
- **Outputs/Return:** None
- **Side effects:** Clamps speed to valid range; writes to I/O ports 0x43 (mode), 0x40 (rate LSB/MSB); disables/restores interrupts
- **Calls:** DisableInterrupts, RestoreInterrupts, outp
- **Notes:** Direct hardware I/O; assumes ISA/XT era 8253 timer at standard ports. Base frequency is 1.192030 MHz.

### TS_SetTimer
- **Signature:** `static long TS_SetTimer(long TickBase)`
- **Purpose:** Calculate 8253 reload value from desired task rate; update clock if needed
- **Inputs:** Desired task rate in Hz (TickBase)
- **Outputs/Return:** Actual timer reload value used
- **Side effects:** Calls TS_SetClockSpeed if calculated speed is faster than current rate (to speed up clock only, never slow down)
- **Calls:** TS_SetClockSpeed
- **Notes:** Formula: `speed = 1192030 / TickBase`. Never downclock to avoid missing faster tasks.

### TS_SetTimerToMaxTaskRate
- **Signature:** `static void TS_SetTimerToMaxTaskRate(void)`
- **Purpose:** Find the fastest (lowest rate) task and reprogram timer to match that speed
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Recalculates optimal timer rate; may call TS_SetClockSpeed; disables/restores interrupts
- **Calls:** DisableInterrupts, RestoreInterrupts, TS_SetClockSpeed
- **Notes:** Called on task removal to avoid unnecessarily fast ticks. Defaults to 0x10000 if no tasks remain.

### TS_Startup
- **Signature:** `static int TS_Startup(void)`
- **Purpose:** Initialize task scheduler on first use; lock memory, allocate stack, hook INT 8
- **Inputs:** None
- **Outputs/Return:** TASK_Ok on success; TASK_Error on allocation or lock failure
- **Side effects:** Calls TS_LockMemory (if LOCKMEMORY); allocates timer stack (if USESTACK); initializes TaskList as circular list; saves original INT 8 vector; hooks TS_ServiceSchedule or TS_ServiceScheduleIntEnabled; sets TS_Installed=TRUE
- **Calls:** TS_LockMemory, allocateTimerStack, _dos_getvect, _dos_setvect, TS_UnlockMemory (on error)
- **Notes:** Idempotent; skipped if TS_Installed already TRUE. Called implicitly by TS_ScheduleTask.

### TS_FreeTaskList
- **Signature:** `static void TS_FreeTaskList(void)`
- **Purpose:** Remove and free all tasks in the list
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Frees all task nodes; resets TaskList to empty circular state; disables/restores interrupts
- **Calls:** DisableInterrupts, RestoreInterrupts, FreeMem
- **Notes:** Marked as TS_LockStart for memory locking region boundary.

### TS_AddTask
- **Signature:** `static void TS_AddTask(task *node)`
- **Purpose:** Insert a new task into the linked list, sorted by descending priority
- **Inputs:** Task node pointer (pre-allocated, fields initialized)
- **Outputs/Return:** None
- **Side effects:** Modifies TaskList structure via LL_SortedInsertion macro
- **Calls:** LL_SortedInsertion macro
- **Notes:** Higher priority values are placed earlier in the list (executed sooner per interrupt).

### TS_LockMemory / TS_UnlockMemory (when LOCKMEMORY defined)
- **Signature:** `int TS_LockMemory(void)` / `void TS_UnlockMemory(void)`
- **Purpose:** Lock/unlock critical code and data in physical memory to prevent page faults during interrupt handling
- **Inputs/Outputs:** LockMemory returns TASK_Ok or TASK_Error; UnlockMemory returns void
- **Side effects:** DPMI calls to memory manager; critical for real-time reliability on systems with virtual memory
- **Calls:** DPMI_LockMemoryRegion, DPMI_Lock, DPMI_UnlockMemoryRegion, DPMI_Unlock
- **Notes:** Locks from TS_LockStart (TS_FreeTaskList) to TS_LockEnd; also locks global state: TaskList, OldInt8, TaskServiceRate, TaskServiceCount, TS_Installed, TS_TimesInInterrupt (if applicable), and stack variables (if USESTACK).

## Control Flow Notes

**Initialization Phase:**
1. TS_ScheduleTask is called to create first task.
2. TS_Startup is invoked (if not already installed): locks memory, allocates stack, initializes TaskList, saves INT 8 vector, hooks interrupt handler.
3. Subsequent TS_ScheduleTask calls add tasks to the list without re-initialization.

**Per-Timer-Tick:**
1. Hardware timer fires INT 8 → TS_ServiceSchedule or TS_ServiceScheduleIntEnabled.
2. Iterate all tasks; increment counters by TaskServiceRate.
3. For each task where `count >= rate`, decrement rate and invoke `TaskService( task )` callback.
4. Update TaskServiceCount; chain to original INT 8 on overflow; send EOI.

**Task Activation:**
- Tasks are created with `active=FALSE`.
- TS_Dispatch walks the list, setting `active=TRUE` for all (typically called once after scheduling).

**Dynamic Rate Changes:**
- TS_SetTaskRate updates a task's rate and recalculates optimal timer clock via TS_SetTimerToMaxTaskRate.

**Task Termination:**
- TS_Terminate removes task from list, frees memory, recalculates optimal clock.
- After TS_Shutdown, list is reset to empty, INT 8 restored, TS_Installed=FALSE.

## External Dependencies

**Headers:**
- `interrup.h`: DisableInterrupts(), RestoreInterrupts() — inline asm for PUSHFD/POPFD/CLI
- `linklist.h`: LL_SortedInsertion, LL_RemoveNode macros — doubly-linked list operations
- `task_man.h`: task struct, public API declarations, TASK_* error codes
- `dpmi.h`: DPMI_LockMemoryRegion(), DPMI_Lock(), etc. — protected-mode memory management
- `usrhooks.h` (optional): USRHOOKS_GetMem, USRHOOKS_FreeMem — custom allocator

**DOS/x86 Functions (defined elsewhere):**
- `_dos_getvect()`, `_dos_setvect()` — interrupt vector table access
- `_chain_intr()` — chain to previous interrupt handler
- `_enable()`, `_disable()` — global interrupt control (POPFD, PUSHFD/CLI)
- `int386()` — execute real-mode interrupt (DPMI)
- `outp()`, `inp()` — I/O port read/write (8253 timer control)
- `malloc()`, `free()` — standard C memory allocation
- `memset()` — memory initialization

**Watcom-specific:**
- `#pragma aux` — inline assembly function declarations (GetStack, SetStack, DPMI calls)

**Hardware:**
- 8253/8254 programmable interval timer at ports 0x40 (data) and 0x43 (control)
- INT 8 (Timer Tick interrupt)
- Base timer frequency: 1.192030 MHz
