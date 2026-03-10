# audiolib/public/timer/source/timer.c

## File Purpose
A demonstration/test program showing how to use the TASK_MAN timer task scheduler. It creates multiple timers running at different rates, dynamically modifies their rates, and shows task termination. This is educational code from the Apogee Software era (1994–1995).

## Core Responsibilities
- Demonstrate task scheduler initialization and usage
- Create and schedule multiple independent timer tasks
- Show dynamic task rate modification via `TS_SetTaskRate`
- Demonstrate task termination via `TS_Terminate`
- Provide a simple timer callback function that increments counters
- Display running timer values in a loop

## Key Types / Data Structures
None defined in this file. Uses `task` struct from task_man.h.

## Global / File-Static State
None.

## Key Functions / Methods

### main
- **Signature:** `void main(void)`
- **Purpose:** Entry point demonstrating task scheduler workflow with three independent timers
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Schedules tasks, calls task dispatcher, modifies task rates, prints to stdout via printf
- **Calls:** `TS_ScheduleTask`, `TS_Dispatch`, `TS_SetTaskRate`, `TS_Terminate`, `TS_Shutdown`, `printf`
- **Notes:** Creates three `task` pointers initialized to NULL. Timer counters are local integers passed as `data` pointers to tasks. Main loop spins on `Timer1 < 300` (approximately 5 seconds at 60 ticks/sec). Demonstrates four phases: start tasks 1–2, add task 3, modify task 2's rate, terminate task 2.

### TimerFunc
- **Signature:** `void TimerFunc(task *Task)`
- **Purpose:** Timer callback function; increments the integer counter passed through the task's data pointer
- **Inputs:** `task *Task` (pointer with `data` member pointing to `int`)
- **Outputs/Return:** None (void)
- **Side effects:** Increments the integer at `Task->data`
- **Calls:** None
- **Notes:** Generic callback; assumes `Task->data` is a valid `int *`. No bounds checking or validation.

## Control Flow Notes
Linear demonstration flow, not a typical game loop:
1. Schedule tasks 1 & 2 with `TS_ScheduleTask`, dispatch with `TS_Dispatch`
2. Wait ~5 sec (spin until Timer1 reaches 300)
3. Schedule task 3, dispatch again
4. Repeat wait
5. Modify task 2's rate to 1000 with `TS_SetTaskRate`
6. Repeat wait
7. Terminate task 2, wait again
8. Terminate tasks 1 & 3, call `TS_Shutdown`

## External Dependencies
- **Standard headers:** `<stdio.h>` (printf), `<stdlib.h>`, `<conio.h>` (console I/O, DOS era), `<dos.h>` (DOS-specific)
- **Local header:** `"task_man.h"` (task struct definition and scheduler API)
- **Defined elsewhere:** `TS_ScheduleTask`, `TS_Dispatch`, `TS_SetTaskRate`, `TS_Terminate`, `TS_Shutdown` (task_man.c)
