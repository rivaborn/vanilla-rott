# audiolib/source/pcfx.c

## File Purpose
Low-level PC speaker sound effects driver for rendering mono audio samples created by Muse. Provides playback control, single-voice voice management, and volume control via the internal PC speaker using I/O port writes. Tightly integrated with the task scheduler for real-time sample output.

## Core Responsibilities
- Manage single-voice mono playback with priority preemption
- Render audio samples to PC speaker via I/O ports (0x61, 0x42, 0x43)
- Support dual sample formats: pitch-indexed lookup table or raw 16-bit PCM samples
- Integrate with task manager to output samples at regular intervals
- Track playback state and handle completion callbacks
- Lock/unlock memory regions for real-time safe operation in protected mode
- Provide error reporting and volume control (0–255 range)

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| PCSound | struct (defined in pcfx.h) | Audio asset: length, priority, and raw sample data array |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| PCFX_Sound | char* | static | Current sound buffer being played; NULL when idle |
| PCFX_LengthLeft | long | static | Remaining samples in current playback |
| PCFX_LastSample | int | static | Last sample value output; cached to minimize I/O |
| PCFX_Lookup | short[256] | static | Pitch lookup table for indexed sample mode |
| PCFX_UseLookupFlag | int | static | Whether current sound uses indexed or raw 16-bit mode |
| PCFX_Priority | int | static | Priority of currently playing sound (preemption control) |
| PCFX_CallBackFunc | void(*)(unsigned long) | static | Optional completion callback; NULL if none |
| PCFX_CallBackVal | unsigned long | static | Callback parameter |
| PCFX_TotalVolume | int | static | Master volume (0–255); mutes speaker when zero |
| PCFX_ServiceTask | task* | static | Handle to repeating service task |
| PCFX_VoiceHandle | int | static | Current voice identifier (incremented on each play) |
| PCFX_Installed | int | global | True if module initialized and task manager is active |
| PCFX_ErrorCode | int | global | Last error status (PCFX_Ok, PCFX_NoVoices, etc.) |

## Key Functions / Methods

### PCFX_Init
- **Signature:** `int PCFX_Init(void)`
- **Purpose:** Initialize the sound effects engine, lock memory, and start the service task.
- **Inputs:** None
- **Outputs/Return:** PCFX_Ok on success; PCFX_Error if DPMI memory locking fails
- **Side effects:** Locks critical functions/data into physical RAM via DPMI; schedules PCFX_Service at 140 Hz; sets PCFX_Installed flag; calls PCFX_Shutdown if already running
- **Calls:** PCFX_LockMemory, PCFX_UseLookup, PCFX_Stop, TS_ScheduleTask, TS_Dispatch
- **Notes:** If called twice, reinitializes; lookup table is populated with pitch ramp (60 units per entry)

### PCFX_Shutdown
- **Signature:** `int PCFX_Shutdown(void)`
- **Purpose:** Halt playback and clean up the sound effects engine.
- **Inputs:** None
- **Outputs/Return:** PCFX_Ok
- **Side effects:** Stops current sound; terminates service task; unlocks all memory regions; clears PCFX_Installed flag
- **Calls:** PCFX_Stop, TS_Terminate, PCFX_UnlockMemory
- **Notes:** Safe to call even if not installed; no-op if PCFX_Installed is false

### PCFX_Play
- **Signature:** `int PCFX_Play(PCSound *sound, int priority, unsigned long callbackval)`
- **Purpose:** Start playback of a sound effect at the specified priority; preempt lower-priority sounds.
- **Inputs:** sound (PCSound struct with length and data); priority (preemption level); callbackval (opaque parameter for completion callback)
- **Outputs/Return:** New voice handle (≥ PCFX_MinVoiceHandle); PCFX_Warning if priority too low
- **Side effects:** Stops any lower-priority sound; increments PCFX_VoiceHandle; disables interrupts during state update
- **Calls:** PCFX_Stop
- **Notes:** Priority < PCFX_Priority returns PCFX_Warning and does not start playback; voice handle wraps at PCFX_MinVoiceHandle

### PCFX_Stop
- **Signature:** `int PCFX_Stop(int handle)`
- **Purpose:** Halt playback of the current sound effect if its handle matches.
- **Inputs:** handle (voice handle to match against PCFX_VoiceHandle)
- **Outputs/Return:** PCFX_Ok if stopped; PCFX_Warning if handle mismatch or no sound playing
- **Side effects:** Clears PCFX_Sound and PCFX_LengthLeft; mutes speaker (I/O 0x61); invokes callback if registered
- **Calls:** PCFX_CallBackFunc (if non-null)
- **Notes:** Disables interrupts during mute operation; callback fired after speaker is muted

### PCFX_Service
- **Signature:** `static void PCFX_Service(task *Task)`
- **Purpose:** Timer callback to output the next audio sample; called ~140 times per second.
- **Inputs:** Task (opaque task manager handle)
- **Outputs/Return:** None
- **Side effects:** Reads next sample from PCFX_Sound; updates speaker frequency via I/O ports 0x43, 0x42; controls speaker enable/disable via I/O 0x61
- **Calls:** PCFX_Stop (when playback exhausted)
- **Notes:** Uses PCFX_LastSample to skip redundant I/O writes (frequency unchanged); in lookup mode, increments by 1 byte; in raw mode, increments by sizeof(short); calls PCFX_Stop when PCFX_LengthLeft reaches zero

### PCFX_VoiceAvailable
- **Signature:** `int PCFX_VoiceAvailable(int priority)`
- **Purpose:** Check if a voice can be preempted at the specified priority.
- **Inputs:** priority (proposed sound priority)
- **Outputs/Return:** TRUE if priority ≥ PCFX_Priority; FALSE otherwise
- **Side effects:** None
- **Notes:** Used by higher-level code to decide whether to attempt playback

### PCFX_SetTotalVolume / PCFX_GetTotalVolume
- Set/get master volume (0–255). SetTotalVolume mutes speaker (I/O 0x61) if volume is zero; disables interrupts during update.

### PCFX_UseLookup
- **Purpose:** Configure pitch lookup table mode. Builds a 256-entry lookup with linear pitch increments (value parameter).
- **Notes:** Stops current playback before reconfiguring.

### PCFX_SetCallBack
- **Purpose:** Register a completion callback (invoked by PCFX_Stop).

### PCFX_LockMemory / PCFX_UnlockMemory
- Lock/unlock all state variables and code section (PCFX_LockStart to PCFX_LockEnd) in physical RAM via DPMI, ensuring real-time service is not delayed by page faults.

### PCFX_ErrorString
- Map error codes to human-readable strings; recursive lookup if error is PCFX_Warning or PCFX_Error.

## Control Flow Notes
**Initialization:** PCFX_Init → PCFX_LockMemory → PCFX_UseLookup → TS_ScheduleTask(PCFX_Service @ 140 Hz)

**Per-frame/sample:** Task manager fires PCFX_Service ~140 times/sec → reads next sample → writes to speaker I/O ports → decrements length counter → calls PCFX_Stop when exhausted (triggers callback)

**Playback request:** PCFX_Play → preempts lower-priority sound via PCFX_Stop → updates state (length, pointer, priority) under interrupt disable → service task picks up on next tick

**Shutdown:** PCFX_Shutdown → PCFX_Stop → TS_Terminate → PCFX_UnlockMemory

## External Dependencies
- **Includes:** `<dos.h>`, `<stdlib.h>`, `<conio.h>` (legacy DOS/Watcom headers)
- **Local headers:** dpmi.h (memory locking), task_man.h (scheduling), interrup.h (interrupt control), pcfx.h (public API & PCSound struct)
- **I/O ports (hardware):** 0x61 (speaker control), 0x42, 0x43 (timer frequency)
- **Symbols defined elsewhere:**
  - DPMI_LockMemoryRegion, DPMI_Lock, DPMI_UnlockMemoryRegion, DPMI_Unlock (memory protection)
  - TS_ScheduleTask, TS_Dispatch, TS_Terminate (task scheduling)
  - DisableInterrupts, RestoreInterrupts (interrupt control)
  - max, min macros (stdlib)
