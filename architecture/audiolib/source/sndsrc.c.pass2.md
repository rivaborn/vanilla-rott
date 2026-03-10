# audiolib/source/sndsrc.c — Enhanced Analysis

## Architectural Role

`sndsrc.c` is a **device driver adapter** for the Disney Sound Source, a parallel-port audio device from the early 1990s. It abstracts low-level hardware I/O (port operations, timer interrupt coordination) behind a simple buffered playback API (`SS_BeginBufferedPlayback`, `SS_StopPlayback`), allowing higher-level audio management subsystems to treat it uniformly alongside other sound devices (Sound Blaster, AdLib, etc.). The file demonstrates interrupt-driven streaming on resource-constrained hardware via cooperative multitasking and circular buffer management.

## Key Cross-References

### Incoming (who depends on this file)
- **High-level audio system** (likely `sndcards.c` or equivalent): Calls `SS_Init`, `SS_BeginBufferedPlayback`, `SS_StopPlayback`, `SS_GetCurrentPos`, `SS_ErrorString` to abstract hardware details
- **Application configuration** (via `user.h`): Reads user-supplied port hints via `USER_CheckParameter()`
- **Task scheduler** (calls into `task_man.h`): Relies on `TS_ScheduleTask`, `TS_Dispatch`, `TS_Terminate` for interrupt timing

### Outgoing (what this file depends on)
- **Task scheduler** (`task_man.h`): `TS_ScheduleTask()` schedules interrupt handler, `TS_Terminate()` stops playback
- **DPMI services** (`dpmi.h`): `DPMI_LockMemoryRegion`, `DPMI_Lock`, `DPMI_Unlock` lock interrupt-sensitive code/data to prevent page faults during real-mode interrupt service
- **User configuration** (`user.h`): `USER_CheckParameter()` checks for command-line port overrides (`SELECT_SOUNDSOURCE_PORT1/2/3`)
- **Hardware definitions** (`sndcards.h`): `TandySoundSource` enum selects between Disney (0x0C) and Tandy (0x0E) command variants
- **Port I/O** (`dos.h`, `conio.h`): `inp()`, `outp()` for direct parallel-port register reads/writes

## Design Patterns & Rationale

1. **Memory-Locked Interrupt Handler** (`SS_ServiceInterrupt` marked with `SS_LockStart`/`SS_LockEnd`):
   - DOS/DPMI era required code & data to be pinned in physical RAM to prevent page faults during interrupt service (crashes the system if violated)
   - Modern OSes handle this automatically; this pattern is a relic of real-mode interrupt handling
   - Critical for reliability on 1990s hardware

2. **Cooperative Timer Interrupt with Yield-Per-Sample**:
   - Limits to 14 samples per tick (`if (count > 13) break;`) to avoid starving main program
   - Non-preemptive approach: interrupt handler drains only what's ready, returns control promptly
   - Reflects DOS-era constraints (no true multitasking, system becomes unresponsive if interrupt blocks)

3. **Circular Buffer with Division-Based Callbacks**:
   - Divides buffer into `NumDivisions` equal chunks; when each chunk is consumed, invokes `SS_CallBack()`
   - Allows application to refill consumed divisions while playback continues (classic double/triple-buffering)
   - Data flow: `SS_BeginBufferedPlayback` → sets up pointers → `SS_ServiceInterrupt` drains → `SS_CallBack()` signals refill

4. **Hardware Detection with User Override**:
   - Tries three standard parallel ports (0x3BC, 0x378, 0x278) via `SS_TestSoundSource()` if user doesn't specify
   - User can skip probes with `SELECT_SOUNDSOURCE_PORTx` parameter (improves startup time)
   - Reflects ISA-era design: multiple interrupt-driven probes were expensive

5. **Variant Device Support** via Parameter:
   - Single `soundcard` parameter to `SS_Init()` switches between Disney (0x0C) and Tandy (0x0E) control commands
   - Minimal abstraction; same core logic works for both with one port value change

## Data Flow Through This File

```
User Code
  ├─ SS_Init(soundcard) → SS_DetectSoundSource() → SS_TestSoundSource() → probe ports
  │                       SS_LockMemory() → DPMI locks interrupt code/data
  │
  ├─ SS_BeginBufferedPlayback(BufferStart, Size, NumDivisions, Callback)
  │  ├─ Initialize buffer state (pointers, sizes, division counters)
  │  └─ TS_ScheduleTask(SS_ServiceInterrupt, 510 ticks/sec) → start interrupt
  │
  ├─ [Interrupt fires ~510 times/sec]
  │  └─ SS_ServiceInterrupt()
  │     ├─ While (port ready for next sample):
  │     │  ├─ outp(port, *SS_SoundPtr++) → write sample byte to DAC
  │     │  ├─ Decrement SS_CurrentLength
  │     │  └─ If division exhausted:
  │     │     ├─ Advance SS_CurrentBuffer, SS_BufferNum (circular)
  │     │     └─ Call SS_CallBack() → user application refills next chunk
  │     └─ Break after 14 samples (yield control)
  │
  └─ SS_StopPlayback() → TS_Terminate(SS_Timer) → write stop command to port
```

**Key invariants:**
- `SS_SoundPtr` advances one byte per port write (one sample at a time)
- `SS_CurrentLength` tracks remaining samples in current division
- `SS_BufferNum` cycles: 0 → NumDivisions-1 → 0 (circular)
- `SS_SoundPlaying` (volatile) signals to main code whether playback is active

## Learning Notes

**Idiomatic to this era / engine:**
- **Real-mode interrupt safety via memory locking**: Modern engines run in protected/long mode; this requirement vanishes. Studying this teaches why interrupt handlers are dangerous in low-level systems.
- **Cooperative scheduling in interrupt handlers**: Pre-emptive multitasking didn't exist; interrupt handlers had to "be kind" and not block. Many game engines (especially 1990s DOS games) had custom scheduler/interrupt hybrid code like this.
- **Circular buffers for streaming**: Still used today in audio, graphics, and networking. The callback-per-division pattern is a precursor to modern ring buffers and audio graph systems.
- **Direct hardware I/O**: Port I/O is how early sound cards worked. Modern audio interfaces (USB, PCIe) use DMA, higher-level abstractions, and the OS kernel handles the driver.
- **Hardware variant handling via parameters**: Reflects limited driver ecosystem (no PnP, ACPI, or device classes; "just pick a port and hope").

**Modern contrast:**
- Today's audio drivers use **asynchronous DMA** (card pulls data, not CPU pushing samples)
- **Kernel-mode execution** with strict memory protection (no locking required)
- **Standard APIs** (ALSA, CoreAudio, WASAPI) abstract hardware completely
- **Plug-and-play detection** (USB enumerates devices automatically)

## Potential Issues

1. **Race condition on `SS_SoundPtr` and `SS_SoundPlaying`**: Declared `volatile` but not atomically protected. If main code checks `SS_SoundPlaying` while interrupt modifies `SS_SoundPtr`, undefined behavior is possible. (However, in practice, pointer reads/writes are atomic on 80386+, so this likely works.)

2. **No error recovery in `SS_ServiceInterrupt`**: Port writes (`outp()`) can fail silently; no way to detect or recover. If hardware is removed, audio just stops with no notification.

3. **Hard-coded timer frequency (510 ticks/sec)**: Not tunable; commented-out 438 Hz variant suggests someone experimented. No public API to adjust playback rate (hardware is fixed at ~7 kHz anyway, but a flexible driver would expose this).

4. **DPMI lock failures silently ignored**: If `SS_LockMemory()` fails, subsequent interrupt service will crash. Real production code should check return status and report errors.

5. **No bounds checking on `SS_SoundPtr`**: Theoretically, if `SS_CurrentLength` wraps or corruption occurs, `SS_SoundPtr` could read beyond `SS_BufferEnd`. The circular buffer logic assumes correct initialization; no runtime verification.

6. **Callback invoked from interrupt context**: User callback runs with interrupts on; if it takes too long, subsequent interrupts may be lost. No timeout or deadlock protection.
