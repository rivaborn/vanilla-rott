# audiolib/source/adlibfx.c

## File Purpose
Low-level Adlib sound card driver for playing sound effects created by the Muse editor. Manages hardware initialization, sound playback control, and task-based audio updates. Handles a single monophonic Adlib voice with volume and priority management.

## Core Responsibilities
- Initialize and shut down the Adlib FX engine with DPMI memory locking
- Manage a single Adlib hardware voice for sound effect playback
- Send low-level register writes to the Adlib card (port 0x388)
- Schedule periodic service routine via task manager to advance playback
- Control sound volume (per-sound and global) with hardware register updates
- Implement priority-based voice stealing (don't play lower-priority sounds over current)
- Support completion callbacks when sounds finish playing

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ALSound` | struct | Adlib sound effect data: FM parameters, sample data pointer, length, priority, block value, modulator/carrier characteristics |
| `task` | struct | Task manager structure for scheduling the service routine |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ADLIBFX_Sound` | `ALSound*` | static | Current sound being played; NULL if idle |
| `ADLIBFX_SoundPtr` | `char*` | static | Current position within the sound data stream |
| `ADLIBFX_LengthLeft` | `long` | static | Bytes remaining to play in current sound |
| `ADLIBFX_Block` | `int` | static | Adlib block register value (frequency/octave) |
| `ADLIBFX_Priority` | `int` | static | Priority of currently playing sound |
| `ADLIBFX_SoundVolume` | `int` | static | Per-sound volume (0–255) |
| `ADLIBFX_TotalVolume` | `int` | static | Global output volume (0–255); default `ADLIBFX_MaxVolume` |
| `ADLIBFX_CallBackFunc` | function pointer | static | Callback invoked when sound completes; NULL if disabled |
| `ADLIBFX_CallBackVal` | `unsigned long` | static | Value passed to callback |
| `ADLIBFX_ServiceTask` | `task*` | static | Handle to scheduled service task |
| `ADLIBFX_VoiceHandle` | `int` | static | Incrementing handle for the current voice |
| `ADLIBFX_Installed` | global `int` | global | TRUE if engine is initialized |
| `ADLIBFX_ErrorCode` | global `int` | global | Last error code |

## Key Functions / Methods

### ADLIBFX_Init
- **Signature:** `int ADLIBFX_Init(void)`
- **Purpose:** Initialize the Adlib FX engine, lock critical code/data in memory, and start the periodic service task.
- **Inputs:** None
- **Outputs/Return:** `ADLIBFX_Ok` on success; `ADLIBFX_Error` if DPMI locking fails.
- **Side effects:** Sets `ADLIBFX_Installed = TRUE`; disables callbacks; schedules `ADLIBFX_Service` task at rate 140 with priority 2; locks code region from `ADLIBFX_SendOutput` to `ADLIBFX_LockEnd` and locks static state variables.
- **Calls:** `ADLIBFX_Shutdown()`, `DPMI_LockMemoryRegion()`, `DPMI_Lock()` (x8), `ADLIBFX_Stop()`, `TS_ScheduleTask()`, `TS_Dispatch()`
- **Notes:** Guards against double-init by calling shutdown first. Memory locking is critical because the service routine may run at interrupt time.

### ADLIBFX_Shutdown
- **Signature:** `int ADLIBFX_Shutdown(void)`
- **Purpose:** Disable the engine, terminate the service task, and unlock memory regions.
- **Inputs:** None
- **Outputs/Return:** `ADLIBFX_Ok`
- **Side effects:** Calls `ADLIBFX_Stop()`; terminates service task; sets `ADLIBFX_Installed = FALSE`; unlocks all locked memory.
- **Calls:** `ADLIBFX_Stop()`, `TS_Terminate()`, `DPMI_UnlockMemoryRegion()`, `DPMI_Unlock()` (x8)
- **Notes:** Safe to call when not installed (early exit).

### ADLIBFX_Play
- **Signature:** `int ADLIBFX_Play(ALSound *sound, int volume, int priority, unsigned long callbackval)`
- **Purpose:** Start playback of a sound effect, initializing all FM operator parameters and carrier level.
- **Inputs:** 
  - `sound`: Pointer to ALSound structure with FM parameters and sample data.
  - `volume`: Requested per-sound volume (0–255).
  - `priority`: Priority level; must be ≥ current priority to play.
  - `callbackval`: User value passed to callback when sound ends.
- **Outputs/Return:** New voice handle (always ≥ `ADLIBFX_MinVoiceHandle`); `ADLIBFX_Warning` if priority too low.
- **Side effects:** Stops current sound; increments `ADLIBFX_VoiceHandle`; disables interrupts; initializes static state (`ADLIBFX_Sound`, `ADLIBFX_SoundPtr`, `ADLIBFX_LengthLeft`, `ADLIBFX_Priority`); writes 11 hardware registers via `ADLIBFX_SendOutput()` (modulator char, scale, attack, sustain, waveform; carrier char, scale, attack, sustain, waveform; connection).
- **Calls:** `ADLIBFX_Stop()`, `DisableInterrupts()`, `ADLIBFX_SendOutput()` (x11), `RestoreInterrupts()`
- **Notes:** Volume calculation blends per-sound and global volume. Carrier level is XORed with 0x3f (inverted) before scaling and then re-inverted.

### ADLIBFX_Service
- **Signature:** `static void ADLIBFX_Service(task *Task)`
- **Purpose:** Task manager callback to advance sound playback; called periodically (140 Hz).
- **Inputs:** `Task`: Pointer to the task structure (unused).
- **Outputs/Return:** None
- **Side effects:** Reads next byte from `ADLIBFX_SoundPtr`; if non-zero, writes to Adlib frequency (0xa0) and block (0xb0) registers; if zero, silences (0xb0 = 0). Decrements `ADLIBFX_LengthLeft`; calls `ADLIBFX_Stop()` when playback complete.
- **Calls:** `ADLIBFX_SendOutput()`, `ADLIBFX_Stop()`
- **Notes:** No explicit interrupt disable; assumes task manager serializes calls. Checks `ADLIBFX_SoundPtr` non-NULL before dereferencing.

### ADLIBFX_Stop
- **Signature:** `int ADLIBFX_Stop(int handle)`
- **Purpose:** Halt playback and clear the current sound state.
- **Inputs:** `handle`: Voice handle to stop; must match current `ADLIBFX_VoiceHandle`.
- **Outputs/Return:** `ADLIBFX_Ok` on success; `ADLIBFX_Warning` if handle mismatch.
- **Side effects:** Disables interrupts; writes 0x00 to Adlib 0xb0 (note off); clears `ADLIBFX_Sound`, `ADLIBFX_SoundPtr`, `ADLIBFX_LengthLeft`, `ADLIBFX_Priority`. If callback is registered, invokes it with `ADLIBFX_CallBackVal`.
- **Calls:** `DisableInterrupts()`, `ADLIBFX_SendOutput()`, `RestoreInterrupts()`, callback function (if set)
- **Notes:** Always succeeds if called unconditionally (valid as a cleanup). Callback may be NULL.

### ADLIBFX_SetVolume
- **Signature:** `int ADLIBFX_SetVolume(int handle, int volume)`
- **Purpose:** Change the volume of the currently playing sound.
- **Inputs:** 
  - `handle`: Voice handle; must match current.
  - `volume`: New per-sound volume (clamped to 0–255).
- **Outputs/Return:** `ADLIBFX_Ok` or `ADLIBFX_Warning` if handle mismatch.
- **Side effects:** Clamps and stores `ADLIBFX_SoundVolume`; recalculates carrier level as in `ADLIBFX_Play()`; writes to register 0x43 (carrier level).
- **Calls:** `DisableInterrupts()`, `ADLIBFX_SendOutput()`, `RestoreInterrupts()`
- **Notes:** Respects global volume in calculation.

### ADLIBFX_SendOutput
- **Signature:** `static void ADLIBFX_SendOutput(int reg, int data)`
- **Purpose:** Write a register value to the Adlib card with proper timing delays.
- **Inputs:** 
  - `reg`: Register address (0x20–0xe3).
  - `data`: Value to write.
- **Outputs/Return:** None
- **Side effects:** Disables interrupts; writes to port 0x388 (register select) and 0x389 (data); busy-waits using `inp()` to satisfy Adlib timing constraints (6 cycles before data write, 35 after).
- **Calls:** `DisableInterrupts()`, `outp()` (x2), `inp()` (x41), `RestoreInterrupts()`
- **Notes:** Hardware-critical function; locked in memory. No protection against concurrent writes beyond interrupt disable.

### ADLIBFX_SoundPlaying
- **Signature:** `int ADLIBFX_SoundPlaying(int handle)`
- **Purpose:** Check if a sound is currently playing.
- **Inputs:** `handle`: Voice handle.
- **Outputs/Return:** TRUE if handle matches current and bytes remain; FALSE otherwise.
- **Calls:** None
- **Notes:** Simple state check; no side effects.

### ADLIBFX_VoiceAvailable
- **Signature:** `int ADLIBFX_VoiceAvailable(int priority)`
- **Purpose:** Determine if a sound at the given priority can be played.
- **Inputs:** `priority`: Requested priority.
- **Outputs/Return:** TRUE if priority ≥ current; FALSE otherwise (preemption check).
- **Calls:** None
- **Notes:** Used for offline priority checks before calling `ADLIBFX_Play()`.

### ADLIBFX_SetTotalVolume
- **Signature:** `int ADLIBFX_SetTotalVolume(int volume)`
- **Purpose:** Set global output volume and recompute carrier level of current sound.
- **Inputs:** `volume`: Global volume (clamped to 0–255).
- **Outputs/Return:** `ADLIBFX_Ok`
- **Side effects:** Clamps and stores `ADLIBFX_TotalVolume`; calls `ADLIBFX_SetVolume()` to update hardware.
- **Calls:** `ADLIBFX_SetVolume()`
- **Notes:** Affects all currently playing sounds (monophonic, so only one possible).

### ADLIBFX_GetTotalVolume
- **Signature:** `int ADLIBFX_GetTotalVolume(void)`
- **Purpose:** Return the current global volume.
- **Outputs/Return:** `ADLIBFX_TotalVolume`
- **Calls:** None

### ADLIBFX_ErrorString
- **Signature:** `char *ADLIBFX_ErrorString(int ErrorNumber)`
- **Purpose:** Map error codes to human-readable strings.
- **Inputs:** `ErrorNumber`: Error code (or -1 for current error).
- **Outputs/Return:** Pointer to static string.
- **Calls:** Recursive call on `ADLIBFX_Warning` or `ADLIBFX_Error`.
- **Notes:** Handles special codes (`ADLIBFX_Warning`, `ADLIBFX_Error`) by redirecting to current error.

### ADLIBFX_SetCallBack
- **Signature:** `void ADLIBFX_SetCallBack(void (*function)(unsigned long))`
- **Purpose:** Register a callback function to invoke when the current sound ends.
- **Inputs:** `function`: Function pointer (or NULL to disable).
- **Outputs/Return:** None
- **Side effects:** Stores function pointer in `ADLIBFX_CallBackFunc`.
- **Notes:** Callback is invoked from `ADLIBFX_Stop()` with `ADLIBFX_CallBackVal`.

### ADLIBFX_LockEnd
- **Signature:** `static void ADLIBFX_LockEnd(void)`
- **Purpose:** Marker function for the end of the memory-locked region.
- **Notes:** Empty function used only to define the boundary for `DPMI_LockMemoryRegion()`.

## Control Flow Notes

**Initialization phase:** `ADLIBFX_Init()` sets up memory locking and schedules the service task at timer interrupt.

**Playback phase:** 
- User calls `ADLIBFX_Play()` → initializes hardware registers and state.
- Periodically (at ~140 Hz), the task manager invokes `ADLIBFX_Service()` → reads the next sample byte and updates frequency.
- User may call `ADLIBFX_SetVolume()` or `ADLIBFX_SetTotalVolume()` to adjust output.

**Termination phase:** When `ADLIBFX_LengthLeft` reaches 0, `ADLIBFX_Service()` calls `ADLIBFX_Stop()`, which silences the hardware and invokes the callback (if registered).

Shutdown phase: `ADLIBFX_Shutdown()` terminates the task and unlocks memory.

## External Dependencies

- **`dpmi.h`**: `DPMI_LockMemoryRegion()`, `DPMI_Lock()`, `DPMI_Unlock()`, `DPMI_UnlockMemoryRegion()` — manage real-mode memory protection in protected mode.
- **`task_man.h`**: `TS_ScheduleTask()`, `TS_Terminate()`, `TS_Dispatch()` — periodic task scheduling.
- **`interrup.h`**: `DisableInterrupts()`, `RestoreInterrupts()` — interrupt flag manipulation (inline asm).
- **`al_midi.h`**: Defines `ADLIB_PORT` (0x388); other Adlib card functions not used here.
- **`adlibfx.h`**: Header declaring public API and `ALSound` struct definition.
- **Standard C**: `<dos.h>`, `<stdlib.h>`, `<conio.h>` for DOS I/O (`outp()`, `inp()`); Watcom-specific.
