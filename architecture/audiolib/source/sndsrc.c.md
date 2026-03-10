# audiolib/source/sndsrc.c

## File Purpose

Low-level driver for the Disney Sound Source, a legacy parallel-port audio device (ca. 1994). Manages buffered playback of 8-bit mono digitized audio at 7 kHz via interrupt-driven I/O port transfers with user callback support.

## Core Responsibilities

- **Hardware detection & initialization**: Locate Sound Source at parallel ports (0x3BC, 0x378, 0x278) with optional Tandy variant support
- **Buffered playback management**: Maintain circular buffer state, track playback position, manage multi-buffer cycling
- **Interrupt-driven transfers**: Service timer interrupts to stream samples to the I/O port at a controlled rate (~438–510 ticks/sec)
- **Callback coordination**: Invoke user function when buffer division transfer completes
- **Memory safety for interrupts**: Lock critical code & data in real memory (DPMI) to prevent page faults during interrupt service
- **Error tracking & reporting**: Provide error codes and human-readable error strings

## Key Types / Data Structures

None (uses only primitives and function pointers; `task` struct defined in task_man.h is opaque here).

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| SS_Installed | static int | static | Initialization guard; prevents multiple inits |
| SS_Port | static int | static | Active I/O port address (0x3BC, 0x378, or 0x278) |
| SS_OffCommand | static int | static | Port control value (0x0C for Disney, 0x0E for Tandy) |
| SS_BufferStart, SS_BufferEnd, SS_CurrentBuffer | static char* | static | Buffer pointers for circular playback |
| SS_BufferNum, SS_NumBuffers | static int | static | Current buffer index and total divisions |
| SS_TotalBufferSize, SS_TransferLength, SS_CurrentLength | static int | static | Buffer sizing and remaining samples to transfer |
| SS_SoundPtr | static char* | static | Current playback position in buffer |
| SS_SoundPlaying | volatile static int | static | Playback-active flag (volatile for interrupt visibility) |
| SS_Timer | static task* | static | Scheduled timer task handle |
| SS_CallBack | static void(*)() | static | User callback invoked per buffer division |
| SS_ErrorCode | static int | static | Last error code returned to caller |

## Key Functions / Methods

### SS_ServiceInterrupt
- **Signature:** `static void SS_ServiceInterrupt(task *Task)`
- **Purpose:** Interrupt handler invoked by timer at playback rate; streams sample data to the sound card I/O port.
- **Inputs:** `Task` (opaque timer task context)
- **Outputs/Return:** None
- **Side effects:** Writes to I/O ports (SS_Port, SS_Port+1, SS_Port+2); advances SS_SoundPtr; decrements SS_CurrentLength; cycles SS_CurrentBuffer and SS_BufferNum; calls SS_CallBack() when division completes
- **Calls:** User callback (SS_CallBack); inp/outp for port I/O
- **Notes:** Memory-locked region (SS_LockStart–SS_LockEnd). Limited to 14 samples per tick to avoid blocking. Volatile state accessed from both main and interrupt context.

### SS_BeginBufferedPlayback
- **Signature:** `int SS_BeginBufferedPlayback(char *BufferStart, int BufferSize, int NumDivisions, void (*CallBackFunc)(void))`
- **Purpose:** Start multi-buffered playback with user callback.
- **Inputs:** `BufferStart` (audio data pointer), `BufferSize` (bytes), `NumDivisions` (circular buffer count), `CallBackFunc` (user callback or NULL)
- **Outputs/Return:** SS_Ok or error code
- **Side effects:** Initializes all buffer state; schedules SS_ServiceInterrupt at ~510 ticks/sec; calls TS_Dispatch()
- **Calls:** SS_StopPlayback (if already playing), SS_SetCallBack, TS_ScheduleTask, TS_Dispatch

### SS_StopPlayback
- **Signature:** `void SS_StopPlayback(void)`
- **Purpose:** End active playback and terminate timer interrupt.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Terminates SS_Timer, writes stop commands to I/O port, clears SS_SoundPlaying flag
- **Calls:** TS_Terminate, outp

### SS_GetCurrentPos
- **Signature:** `int SS_GetCurrentPos(void)`
- **Purpose:** Return current playback offset within the active buffer division.
- **Inputs:** None
- **Outputs/Return:** Offset (bytes) or SS_Warning if no sound playing
- **Side effects:** Sets SS_ErrorCode if not playing
- **Calls:** SS_SetErrorCode (macro)

### SS_Init
- **Signature:** `int SS_Init(int soundcard)`
- **Purpose:** Initialize driver: detect hardware, lock memory, configure for Disney or Tandy variant.
- **Inputs:** `soundcard` (TandySoundSource or other to select command variant)
- **Outputs/Return:** SS_Ok or error code
- **Side effects:** Calls SS_DetectSoundSource, SS_LockMemory, sets SS_Installed flag
- **Calls:** SS_DetectSoundSource, SS_LockMemory, outp

### SS_DetectSoundSource
- **Signature:** `int SS_DetectSoundSource(void)`
- **Purpose:** Locate Sound Source by checking user parameters or probing all three parallel ports.
- **Inputs:** None
- **Outputs/Return:** TRUE if found, FALSE otherwise
- **Side effects:** Updates SS_Port on success
- **Calls:** USER_CheckParameter, SS_TestSoundSource

### SS_TestSoundSource
- **Signature:** `int SS_TestSoundSource(int port)`
- **Purpose:** Probe a single port for Sound Source hardware presence.
- **Inputs:** `port` (0x3BC, 0x378, or 0x278)
- **Outputs/Return:** TRUE if detected, FALSE otherwise
- **Side effects:** I/O port writes; schedules and terminates a timer task for delay
- **Calls:** TS_ScheduleTask, TS_Dispatch, TS_Terminate, inp/outp

### SS_Shutdown
- **Signature:** `void SS_Shutdown(void)`
- **Purpose:** Halt playback and release all resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls SS_StopPlayback, writes port stop commands, unlocks memory, clears SS_Installed
- **Calls:** SS_StopPlayback, SS_UnlockMemory, outp

### SS_LockMemory / SS_UnlockMemory
- **Purpose:** Lock/unlock critical code and data in real memory for interrupt safety (DPMI).
- **Inputs/Outputs:** SS_Ok on success, SS_Error on failure
- **Calls:** DPMI_LockMemoryRegion, DPMI_Lock (macro) for each static variable

## Control Flow Notes

**Initialization phase:** User calls SS_Init → detects hardware → locks interrupt-sensitive code and variables in real memory.

**Playback phase:** User calls SS_BeginBufferedPlayback → schedules timer interrupt at ~510 Hz → TS_Dispatch() starts interrupt service → SS_ServiceInterrupt fires, drains current buffer division, cycles to next division, calls user callback.

**Shutdown phase:** SS_StopPlayback terminates timer, SS_Shutdown unlocks memory and marks driver offline.

The design is interrupt-driven; volatile state (SS_SoundPlaying, SS_SoundPtr) is accessed from both main code and the interrupt handler.

## External Dependencies

- **dos.h, conio.h**: `inp()`, `outp()` for legacy parallel-port I/O
- **dpmi.h**: `DPMI_LockMemoryRegion`, `DPMI_Lock`, `DPMI_Unlock` for real-mode interrupt safety (prevents page faults)
- **task_man.h**: `task` struct, `TS_ScheduleTask`, `TS_Terminate`, `TS_Dispatch` for timer interrupt scheduling
- **sndcards.h**: `TandySoundSource` enum for variant selection
- **user.h**: `USER_CheckParameter()` for configuration parameter lookup
