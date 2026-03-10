# audiolib/source/pcfx.c — Enhanced Analysis

## Architectural Role

PCFX is the lowest-tier audio device driver in a multi-layered sound subsystem, implementing fallback PC speaker synthesis for DOS systems lacking dedicated sound hardware (AdLib, Sound Blaster, GUS). While crude by modern standards, it provides essential monaural sound effects via timer-modulated speaker frequency. Its simplicity and universal availability make it the baseline compatibility layer in what appears to be a pluggable driver architecture (mirrored by `adlibfx.c`, `blaster.c`, etc.).

## Key Cross-References

### Incoming (who depends on this file)

From the cross-reference map, higher-level audio subsystems and game code in `rott/rt_*.c` would call this driver's public API:
- `PCFX_Init`, `PCFX_Shutdown` (lifecycle)
- `PCFX_Play`, `PCFX_Stop`, `PCFX_SoundPlaying` (playback control)
- `PCFX_SetTotalVolume`, `PCFX_GetTotalVolume` (volume)
- `PCFX_ErrorString` (diagnostics)

Likely orchestrated by a higher-level audio abstraction (not visible in this excerpt) that selects which driver to activate.

### Outgoing (what this file depends on)

- **Task manager** (`task_man.h`): `TS_ScheduleTask`, `TS_Dispatch`, `TS_Terminate` — drives playback at 140 Hz
- **Interrupt control** (`interrup.h`): `DisableInterrupts`, `RestoreInterrupts` — ensures atomic state updates
- **Protected-mode memory** (`dpmi.h`): `DPMI_LockMemoryRegion`, `DPMI_Lock`, `DPMI_Unlock` — locks hot path in physical RAM
- **DOS/BIOS hardware** (`dos.h`, `conio.h`): Direct I/O port access (0x61 speaker, 0x42/0x43 timer)

## Design Patterns & Rationale

**Single-voice preemption model**: Unlike multi-voice FM/PCM drivers, PCFX treats audio as a single priority-based stream. New `PCFX_Play` calls preempt running sounds if priority is higher. This drastically reduces complexity and latency but sacrifices polyphony.

**Memory locking in DPMI**: The entire service path (PCFX_LockStart → PCFX_LockEnd) is locked, preventing page faults during timer-driven playback. This is a direct consequence of real-time constraints on DOS protected mode (no preemption, no scheduler).

**Sample lookup caching** (`PCFX_LastSample`): Avoids redundant I/O writes if the frequency hasn't changed—a critical optimization when I/O dominates CPU time on the PC speaker.

**Dual sample format** (lookup vs raw 16-bit): The lookup mode compresses 8-bit indices into a pitch table, trading dynamic range for smaller footprint. Raw mode supports full 16-bit PCM but requires 2× memory/bandwidth. The mode is baked in at playback time, not per-sample.

## Data Flow Through This File

1. **Setup** (one-time): `PCFX_Init` → lock memory, populate 256-entry pitch lookup (60 units/step), schedule service task
2. **Playback request**: Game calls `PCFX_Play(sound, priority, callback)` → preempts lower-priority sound → updates PCFX_Sound pointer and length counter (atomic via interrupt disable)
3. **Per-sample** (every ~7ms at 140 Hz): Task manager invokes `PCFX_Service` → read next sample byte (lookup) or short (raw) → if volume > 0 and pitch changed, reprogram timer (I/O 0x43/0x42) and enable speaker (I/O 0x61) → decrement counter
4. **Completion**: Length counter reaches zero → `PCFX_Stop` mutes speaker → fires optional callback
5. **Shutdown**: `PCFX_Shutdown` → terminate task, unlock memory

## Learning Notes

**Era-specific techniques**: This code exemplifies late-DOS sound driver patterns (pre-1996):
- No DMA, no interrupts—polling via cooperative task scheduler
- Direct hardware port access instead of BIOS calls (faster, but platform-specific)
- Manual memory locking for real-time safety (abstraction over paging)
- Lookup tables to avoid division/multiplication in the hot path

**Contrast with modern engines**:
- Modern engines use audio callbacks with garbage-collected sample buffers
- PCFX tightly couples sample format (lookup vs raw) to playback mode, whereas modern drivers abstract this
- No sample rate conversion; output is hardcoded to ~140 Hz (limited by PC speaker duty cycle)
- Single voice is a severe limitation; modern fallbacks (system buzzer, SDL) still support mixing

**Game engine lesson**: This driver is likely selected by runtime hardware detection. The parallel `adlibfx.c` and `blaster.c` drivers share the same API contract, making PCFX a drop-in compatibility fallback—a good example of the strategy pattern in low-level audio.

## Potential Issues

- **No thread safety**: The callback (`PCFX_CallBackFunc`) is invoked synchronously from `PCFX_Stop`, which runs in the task scheduler context. If the caller attempts to reinitialize or play new audio from the callback, race conditions are possible (though unlikely given DOS single-threading).
- **Voice handle wraparound**: `PCFX_VoiceHandle` wraps at `PCFX_MinVoiceHandle` but no guard against collision if a stale handle is reused before cleanup. Unlikely to occur in practice.
- **Lookup table assumption**: `PCFX_UseLookup` is called unconditionally in `PCFX_Init` with hardcoded pitch increment (60). There's no validation that subsequent `PCFX_Play` calls respect the format; mismatched format could cause buffer overruns.
