# audiolib/source/adlibfx.c — Enhanced Analysis

## Architectural Role

ADLIBFX is a driver-level subsystem within a multi-driver audio architecture (alongside AL_MIDI for MIDI synthesis, BLASTER for PCM playback, and AWE32 for wavetable). It provides monophonic FM synthesis for Adlib/OPL2 sound effects created by the Muse editor. The module integrates with the task manager for real-time, interrupt-safe playback scheduling, and with DPMI for memory protection in protected-mode DOS—essential for predictable timing in a game engine running alongside other interrupt handlers.

## Key Cross-References

### Incoming (who depends on this file)
- **Higher-level sound manager** (not visible in excerpt): Would call `ADLIBFX_Play()`, `ADLIBFX_SetVolume()`, `ADLIBFX_Stop()`, `ADLIBFX_Init()`, `ADLIBFX_Shutdown()` to initiate and control sound playback.
- **Game engine subsystems**: May query `ADLIBFX_SoundPlaying()` to decide if a sound is active, or `ADLIBFX_VoiceAvailable()` for priority preemption logic before calling `ADLIBFX_Play()`.

### Outgoing (what this file depends on)
- **Task manager** (`task_man.h`): Schedules `ADLIBFX_Service()` at ~140 Hz via `TS_ScheduleTask()`, `TS_Dispatch()`; terminates via `TS_Terminate()`.
- **DPMI subsystem** (`dpmi.h`): Locks critical code region (`ADLIBFX_SendOutput`...`ADLIBFX_LockEnd`) and all mutable state variables to prevent paging during hardware I/O.
- **Interrupt management** (`interrup.h`): Disables/restores CPU flags around hardware writes via `DisableInterrupts()` / `RestoreInterrupts()`.
- **Hardware (DOS I/O)**: Direct port I/O to Adlib card (port 0x388/0x389) via `outp()` / `inp()`.

## Design Patterns & Rationale

**Singleton monophonic voice**: Only one `ADLIBFX_Sound` active at a time, with no voice pool or multi-channel mixer. This simplifies state management and reflects the Muse editor's limitation (single FM channel per file).

**Handle-based voice identity**: `ADLIBFX_VoiceHandle` increments on each new play, wrapping at `ADLIBFX_MinVoiceHandle`. This prevents a new sound from being accidentally stopped by an old handle reference, though the design assumes short-lived handles and doesn't truly validate ownership.

**Priority-based preemption**: Before playing, `ADLIBFX_VoiceAvailable()` checks if the new priority is ≥ current. This avoids complex voice allocation; the game logic decides what to play based on priority tier.

**Real-time task scheduling + memory locking**: `ADLIBFX_Service()` runs at fixed 140 Hz, with all critical code and data pinned in physical memory via DPMI. This ensures predictable latency and no page-fault interruptions during hardware access—essential on a DOS system with paging.

**Register-level hardware abstraction**: `ADLIBFX_SendOutput()` encapsulates Adlib timing (6-cycle delay before data write, 35-cycle delay after) and interrupt safety, so callers don't replicate this logic.

**Volume blending**: Per-sound volume and global volume are blended (`volume * ADLIBFX_TotalVolume / ADLIBFX_MaxVolume`), then mapped to carrier envelope attenuation via inverted XOR arithmetic. This is typical FM synthesis but non-obvious without Adlib register documentation.

## Data Flow Through This File

**Initialization**: `ADLIBFX_Init()` → locks memory region + state variables → schedules periodic service task → ready for playback.

**Sound playback**: 
1. Caller invokes `ADLIBFX_Play(sound, volume, priority, callback)` 
2. Priority check: if priority < current, reject with `ADLIBFX_NoVoices`
3. Stop any running sound, increment handle
4. Disable interrupts, initialize playback state (`ADLIBFX_Sound`, `ADLIBFX_SoundPtr`, `ADLIBFX_LengthLeft`, `ADLIBFX_Priority`)
5. Write 11 Adlib registers (modulator & carrier FM params)
6. Restore interrupts, return handle

**Periodic playback advance**:
1. Task manager calls `ADLIBFX_Service()` at ~140 Hz
2. Read next byte from `ADLIBFX_SoundPtr` (sample data is frequency/note info)
3. Write to Adlib frequency register (0xa0) and block (0xb0) if non-zero, else silence
4. Decrement `ADLIBFX_LengthLeft`
5. If done, call `ADLIBFX_Stop()` → invoke callback → return

**Volume control**: User calls `ADLIBFX_SetVolume()` → recalculates carrier envelope attenuation → writes single Adlib register (0x43).

**Shutdown**: `ADLIBFX_Shutdown()` → terminates service task → unlocks all memory.

## Learning Notes

**FM synthesis register programming**: Adlib/OPL2 requires programming modulator (0x20/0x40/0x60/0x80/0xe0) and carrier (0x23/0x43/0x63/0x83/0xe3) operator parameters separately. The carrier level calculation (`carrierlevel ^= 0x3f; carrierlevel *= ...; carrierlevel ^= 0x3f`) is an inversion technique to linearize the exponential-scale envelope attenuation register—typical of 1990s FM synth programming.

**Real-time DOS constraints**: This code reflects DOS/protected-mode realities: interrupt disabling, DPMI memory locking, task scheduling, and direct hardware I/O. Modern engines use DMA, buffering, and OS-managed threading; this approach assumes the game loop is the dominant consumer and I/O is synchronous.

**Monophonic simplicity vs. polyphony**: Unlike AL_MIDI (which supports multiple voices), ADLIBFX is intentionally single-voice. This works because Muse sound effects are monophonic FM sequences, not polyphonic instruments. The priority system is sufficient for game logic (e.g., higher-priority sounds preempt ambient effects).

**No implicit resource cleanup**: Sounds must be explicitly stopped or play to completion; there's no auto-release or timeout. This is safe for games that control playback tightly, but risky if sounds are orphaned.

## Potential Issues

**Weak handle validation**: `ADLIBFX_Stop()` and `ADLIBFX_SetVolume()` check handle == current, but don't prevent use-after-free if a handle is reused after wrapping.

**Unprotected global state access**: `ADLIBFX_Service()` reads from multiple globals (`ADLIBFX_Sound`, `ADLIBFX_SoundPtr`, `ADLIBFX_LengthLeft`) without explicit locking. Only interrupt disabling during `ADLIBFX_Play()` protects these writes. If the game logic calls `ADLIBFX_Play()` while the service routine is mid-read, a partial write could corrupt playback state (though DPMI locking should minimize this window).

**Carrier level calculation edge cases**: The inverted XOR arithmetic with `volume` scaled by `ADLIBFX_TotalVolume` could over/underflow if `volume` or scale fields are extreme. No bounds checking on intermediate values.

**No detection of invalid sound pointers**: `ADLIBFX_Play()` doesn't validate the `ALSound*` pointer; a corrupt pointer would cause a crash during `TS_Dispatch()` or the first service call.

**No timeout or overflow check on handle**: `ADLIBFX_VoiceHandle` increments indefinitely; if it wraps to `ADLIBFX_MinVoiceHandle`, old pending stop calls might match a new sound.
