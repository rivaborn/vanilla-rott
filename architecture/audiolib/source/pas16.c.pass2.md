# audiolib/source/pas16.c — Enhanced Analysis

## Architectural Role
PAS16 is a low-level hardware abstraction for Pro AudioSpectrum sound cards, providing DMA-based audio playback/record with interrupt-driven callbacks. It bridges protected-mode game code to DOS real-mode hardware via the MVSOUND.SYS driver, using DPMI for mode transitions and memory locking. Part of a larger multi-backend audio abstraction layer (alongside BLASTER, ADLIB, GUS, AWE32) selected at runtime, implementing a standardized interface (`Init`, `Shutdown`, `BeginBufferedPlayback`, `SetPlaybackRate`, `SetMixMode`).

## Key Cross-References

### Incoming (who depends on this file)
- **Audiolib abstraction layer**: Higher-level code dispatches through a runtime driver-selection mechanism; PAS_* functions are exported in parallel with BLASTER_*, ADLIB_*, GUS_*, AWE32_* (per cross-ref). Suggests a vtable or switch statement selecting the active backend.
- **Game/application code**: Calls `PAS_Init` once at startup, then `PAS_BeginBufferedPlayback` to stream audio with periodic `PAS_GetCurrentPos` queries during playback.
- **Real-mode driver**: Callback function pointer `PAS_CallBack` invoked by `PAS_ServiceInterrupt` to signal buffer refill opportunities.

### Outgoing (what this file depends on)
- **MVSOUND.SYS** (real-mode DOS driver): Accessed via interrupt 0x?? to query state table, function table, DMA/IRQ settings, and call mixer functions (`PAS_CheckForDriver`, `PAS_GetStateTable`, `PAS_GetFunctionTable`, `PAS_GetCardSettings`, `PAS_CallMVFunction`).
- **DMA subsystem** (`dma.h/c`): `DMA_SetupTransfer`, `DMA_GetCurrentPos`, `DMA_EndTransfer` for hardware DMA control.
- **Interrupt management** (`interrup.h`, `irq.h`): `DisableInterrupts`, `RestoreInterrupts`, `_dos_setvect`, `IRQ_SetVector` for safe interrupt installation and unmasking.
- **DPMI layer** (`dpmi.h`): `DPMI_LockMemoryRegion`, `DPMI_CallRealModeFunction` for memory locking and real-mode function invocation.
- **C runtime** (`dos.h`, `conio.h`): `inp`/`outp` for I/O port access, `_dos_getvect`/`_dos_setvect` for vector table manipulation.

## Design Patterns & Rationale

**Protected-Mode ISR with Stack Switching**: Uses `GetStack`/`SetStack` pragma to allocate a dedicated conventional-memory stack for interrupt handling. Rationale: interrupt may fire while user code executes with a minimal stack (e.g., tight render loop), so ISR must not overflow the application's stack.

**Real-Mode Driver Indirection via DPMI**: Rather than directly programming write-only PAS registers, queries a real-mode `MVSOUND.SYS` driver via DPMI. Rationale: Centralizes hardware state caching, allows variant compatibility without recompilation, and simplifies protected-mode logic.

**Circular DMA Buffer with Interrupt-Driven Position Tracking**: Divides application buffer into chunks, DMA loops automatically, ISR advances `PAS_CurrentDMABuffer` pointer and invokes user callback. Rationale: Allows asynchronous refilling; low CPU overhead during playback.

**Stateful Hardware Configuration**: Saves/restores mixer volumes, interrupt masks, and control registers across init/shutdown. Rationale: Essential for clean DOS driver behavior; leaves system in consistent state for other drivers.

## Data Flow Through This File

- **Init phase**: `PAS_Init` → queries MVSOUND.SYS → saves state/volumes → locks memory → allocates ISR stack → installs interrupt vector.
- **Playback phase**: `PAS_BeginBufferedPlayback` → `PAS_SetMixMode`/`PAS_SetPlaybackRate` → `PAS_SetupDMABuffer` → `PAS_BeginTransfer` → **Hardware DMA loops buffer chunks** → Each chunk completion fires ISR → `PAS_ServiceInterrupt` advances position pointer → calls user callback → repeats until `PAS_StopPlayback`.
- **Shutdown phase**: `PAS_Shutdown` → halts DMA → restores interrupt vector → restores mixer volumes → unlocks memory → deallocates stack.

## Learning Notes

**DOS/Watcom C idioms**: Uses `pragma aux` for inline assembly, `union REGS`/`int386` for BIOS calls, and DPMI for protected/real mode bridging—patterns unique to 1990s DOS development. Memory locking (`DPMI_LockMemoryRegion`) is necessary because real-mode ISRs cannot page-fault.

**Write-Only Register Workaround**: PAS cards expose write-only registers; this design offloads state caching to the real-mode driver instead of maintaining a duplicate shadow register file locally.

**Modern contrast**: Contemporary audio engines (ALSA, PulseAudio, Core Audio) hide buffer management and use OS-managed callbacks; they don't require raw interrupt handlers or memory locking.

**Multi-backend architecture**: The parallel structure of PAS_*, BLASTER_*, ADLIB_*, GUS_* functions suggests a runtime-selectable audio backend design—likely controlled by a higher-level dispatcher based on detected hardware.

## Potential Issues

1. **Commented-out state restoration** (~`PAS_RestoreState()` in `PAS_Shutdown`): Hardware registers are not restored, leaving the card in an inconsistent state if unloaded and reloaded. Likely a DEBUG flag.

2. **MVSOUND.SYS hard dependency**: No fallback if the real-mode driver is missing; entire subsystem fails. Abstraction layer should attempt other backends on failure.

3. **No interrupt contention detection**: `PAS_Interrupts` mapping and `VALID_IRQ()` checks exist, but code doesn't probe whether another driver has already claimed the IRQ.

4. **Stack overflow risk if USESTACK disabled**: Without custom ISR stack, interrupt on a shallow user stack could corrupt memory. Mitigated in practice by gating with `#ifdef USESTACK`, but no runtime assertion.
