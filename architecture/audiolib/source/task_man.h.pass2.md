# audiolib/source/task_man.h — Enhanced Analysis

## Architectural Role

This header defines the interrupt-driven task scheduler that forms the temporal backbone of the audio library subsystem. It enables periodic execution of audio service routines (like `ADLIBFX_Service`, `BLASTER_ServiceInterrupt`) without blocking the main game loop, by executing them from timer interrupts at configurable rates and priorities. The scheduler bridges hardware timer interrupts and high-level audio device drivers, allowing them to coordinate mixed-priority, mixed-frequency work.

## Key Cross-References

### Incoming (who depends on this file)
- **Audio device drivers** in audiolib (ADLIBFX, BLASTER, GUS MIDI): These modules register their `*_Service()` callbacks via `TS_ScheduleTask()` to receive periodic invocations from the interrupt handler
- **Game engine main loop** (likely `rott/rt_main.c` or similar): Calls `TS_Dispatch()` to process ready tasks each frame/tick
- **Interrupt handler** (hardware timer ISR): Invokes `TS_Dispatch()` directly or sets a flag to trigger dispatch

### Outgoing (what this file depends on)
- **No explicit dependencies visible** in this header (it's a pure interface)
- Implicitly depends on hardware timer interrupt delivery (assumed provided by platform layer)
- Memory locking API (`TS_LockMemory`, `TS_UnlockMemory`) suggests underlying memory protection mechanism (likely DOS/386 protected mode specifics)

## Design Patterns & Rationale

**Interrupt-Driven Cooperative Scheduler**: Tasks are stored in a linked list (via `task.next`/`task.prev`), each with a countdown timer. `TS_Dispatch()` iterates the list, decrements `count` fields, and invokes callbacks when `count` reaches zero. This allows:
- **Priority scheduling**: Tasks processed in queue order (higher priority enqueued first?)
- **Rate limiting**: `rate` field controls execution frequency; `count` prevents runaway callback invocations
- **Dual-context execution**: `TS_InInterrupt` flag allows shared code (audio drivers) to detect whether they're running in interrupt or normal context, adjusting synchronization accordingly
- **Memory safety in interrupts**: `TS_LockMemory()`/`TS_UnlockMemory()` protect heap operations during interrupt-unsafe DOS mode

**Why this design?** DOS-era games needed real-time audio without blocking the game loop. Hardware timer interrupts provide guaranteed timing; a callback-based queue avoids hard-coded ISR implementations.

## Data Flow Through This File

1. **Registration phase** (init): Audio driver calls `TS_ScheduleTask(service_fn, rate, priority, data)` → task node created and inserted into queue
2. **Dispatch phase** (per frame/interrupt): 
   - Timer interrupt fires → calls `TS_Dispatch()`
   - For each task: `count--`; if `count <= 0` then invoke `task->TaskService(task)` and reset `count = rate`
3. **Teardown phase** (shutdown): `TS_Terminate(task_ptr)` unlinks and deallocates task from queue
4. **Dynamic adjustment**: `TS_SetTaskRate()` updates execution frequency mid-stream

Data pointer (`task->data`) allows audio drivers to pass module-specific state (e.g., device registers, buffer pointers) to their callbacks without global variables.

## Learning Notes

**Idiomatic to 1990s DOS game engines**:
- Callback-based dispatch predates modern event systems and message queues
- Rate-based execution (tick counters) predates delta-time updates common in modern engines
- `volatile int TS_InInterrupt` is a primitive synchronization flag; modern systems use atomics, memory barriers, or proper mutex/semaphore primitives
- Direct memory locking for DOS protected mode is now obsolete (no protected mode, no interrupt-unsafe malloc)

**What this reveals about ROTT's architecture**:
- Audio subsystem was designed for responsive real-time I/O (necessary for 1990s FM synthesis / Sound Blaster cards)
- Game loop and audio I/O operate in two contexts (main loop + interrupt), requiring careful coordination
- No thread abstraction; instead, explicit interrupt context detection (`TS_InInterrupt`) guides code behavior

## Potential Issues

- **Reentrancy risk**: If `TS_ScheduleTask()` or `TS_Terminate()` are called while `TS_Dispatch()` is executing (inside an interrupt), the linked list may be corrupted. Not inferable whether callers respect this invariant.
- **Priority inversion**: If a high-priority task depends on memory locked by a lower-priority task's callback, deadlock could occur. Depends on how audio drivers use `TS_LockMemory()`.
- **No thread safety on `volatile int TS_InInterrupt`**: Multiple CPUs or signal handlers could race to update this flag (unlikely on DOS/i386, but architecturally unsafe).
