# audiolib/source/dpmi.c — Enhanced Analysis

## Architectural Role

This file provides a low-level DPMI abstraction layer that acts as a bridge between the protected-mode audio subsystems (Sound Blaster, Adlib, GUS, AWE32) and real-mode DOS services. It handles three critical functions: managing real-mode interrupt vectors (for audio hardware callbacks), executing real-mode procedures, and locking DMA-critical memory to prevent virtual memory paging. Without this layer, DMA-based audio transfers would fail due to unpredictable memory relocation.

## Key Cross-References

### Incoming (who depends on this file)
- **BLASTER subsystem** (`audiolib/source/blaster.c`): Calls `DPMI_LockMemory()` to lock DMA ring buffers during `BLASTER_Init()`; likely calls `DPMI_SetRealModeVector()` to install interrupt handlers
- **GUS subsystem** (`audiolib/source/gus.c`): Locks sample memory and DMA buffers using `DPMI_LockMemory()`
- **AWE32 subsystem** (`audiolib/source/awe32.c`): May lock wave ROM and effect memory regions
- **AL_MIDI subsystem** (`audiolib/source/al_midi.c`): Locking requirements for MIDI-related real-mode callbacks
- **Audio initialization chain**: All hardware detection and setup routines depend on DPMI services being available early in engine startup

### Outgoing (what this file depends on)
- **`<dos.h>` DPMI primitives**: `int386()`, `int386x()` — execute BIOS interrupt 0x31 with register preservation
- **`<dos.h>` register macros**: `FP_SEG()`, `FP_OFF()` for segment:offset pointer decomposition
- **`dpmi.h`**: Defines `dpmi_regs` struct (CPU register state), `DPMI_Errors` enum
- **DPMI BIOS Service 0x31**: The actual protected-to-real-mode interface provided by DOS/4GW extender

## Design Patterns & Rationale

**Wrapper Pattern**: DPMI functions mask the register-manipulation complexity of BIOS interrupt calls. Callers pass semantic arguments (address, length) rather than manipulating EAX/BX/CX/SI/DI registers directly.

**Static Register Cache**: Both `Regs` and `SegRegs` are file-static, not passed as parameters. This reduces call overhead and matches the register-centric nature of DPMI. Tradeoff: non-reentrant (problematic in async/multi-threaded contexts, though DOS is fundamentally single-threaded).

**Convenience Wrapper Pairs**: `DPMI_LockMemory()` + `DPMI_LockMemoryRegion()`, and `DPMI_UnlockMemory()` + `DPMI_UnlockMemoryRegion()`. The region versions compute byte offsets from pointer pairs, hiding pointer arithmetic from callers.

**32-Bit Segment:Offset Packing**: Real-mode vectors are packed as `(segment << 16) | offset` for convenience, then unpacked into register pairs for DPMI calls. This matches DOS-era address conventions.

**Error Handling via Carry Flag**: DPMI returns errors via the carry flag (standard BIOS convention). All functions check `Regs.x.cflag` and return `DPMI_Ok` / `DPMI_Error` consistently.

## Data Flow Through This File

```
Init Phase:
  Audio subsystem init → DPMI_LockMemory(dma_buffer)
                      → DPMI_SetRealModeVector(interrupt_num, handler)
                      → Real-mode interrupt handler configured

Runtime:
  DMA proceeds on locked memory (no page faults)
  Real-mode callback may invoke DPMI_CallRealModeFunction()

Shutdown:
  Audio subsystem cleanup → DPMI_UnlockMemory(dma_buffer)
                         → Original vectors restored
```

**Address Translation**: Pointer → linear address (DOS/4GW flat model) → split into 16-bit register pairs for DPMI → BIOS call.

## Learning Notes

**DOS Protected-Mode Era (1994–1995)**: This code exemplifies how protected-mode extenders (DOS/4GW) bridge the gap between modern 32-bit code and legacy real-mode hardware. Modern OSes abstract this entirely; here it's explicit.

**Segment:Offset vs. Linear Addressing**: The conversion from C pointers to linear addresses (`linear = (unsigned) address`) is trivial in DOS/4GW's flat memory model but would fail on segmented architectures (i.e., DOS with small/medium memory models).

**DMA Safety Requirement**: Memory locking is non-negotiable for audio. The virtual memory manager cannot know that a DMA controller is accessing a buffer—it might page the buffer out, causing silent data corruption. This pattern appears throughout legacy audio APIs.

**Register State Preservation**: DPMI function 0x0301 (`DPMI_CallRealModeFunction`) passes a `dpmi_regs` block so the caller can provide input registers and receive output registers from the real-mode function. This is more sophisticated than simple interrupt wrappers.

**Idiomatic to This Engine**: The file-static `Regs` / `SegRegs` pattern is typical of 1990s C drivers where global state was acceptable and performance critical. Modern code would encapsulate this in a context struct.

## Potential Issues

1. **Non-Reentrant Static State**: If any audio subsystem attempted concurrent DPMI calls (e.g., via interrupt handler and main thread), they would corrupt each other's register state. Mitigated by DOS's single-threaded nature, but fragile if code is ported.

2. **Unused Include**: `#include <string.h>` is present but no string functions are used. Minor code smell, suggests file was templated or includes were copied without pruning.

3. **No Address Validation**: `DPMI_LockMemoryRegion()` computes length as `(char*)end - (char*)start` with no check for `end < start`. If called with reversed pointers, length becomes a huge unsigned wrap-around value, potentially locking invalid memory.

4. **Unsigned Overflow in Address Splitting**: The `(linear >> 16)` and `(linear & 0xFFFF)` operations assume linear addresses fit in 32 bits. In DOS/4GW this is guaranteed, but the code has no defensive checks.
