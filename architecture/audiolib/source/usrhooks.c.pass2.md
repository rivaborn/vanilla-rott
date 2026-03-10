# audiolib/source/usrhooks.c — Enhanced Analysis

## Architectural Role

This file implements the **memory allocation hook point** for the audio library, sitting at the boundary between `audiolib` and the calling application (ROTT engine). It abstracts malloc/free behind a standardized interface, allowing the host program to customize, monitor, or replace memory allocation behavior without modifying the audio library itself. All audio subsystems (BLASTER, AL_MIDI, ADLIBFX, AWE32) that need dynamic memory ultimately route through these functions.

## Key Cross-References

### Incoming (who depends on this file)
- **Entire audiolib suite**: All hardware driver modules (blaster.c, al_midi.c, adlibfx.c, awe32.c, gus.c, gusmidi2.c, dma.c) would call `USRHOOKS_GetMem` / `USRHOOKS_FreeMem` for:
  - Audio buffer allocation
  - Voice/channel state structures
  - Timbre bank data
  - DMA descriptors
- The calling program (ROTT engine) can provide its own `usrhooks.h` declarations to intercept or customize allocation

### Outgoing (what this file depends on)
- `<stdlib.h>` — C standard library malloc/free
- `usrhooks.h` — Exports the two functions and error code enum (`USRHOOKS_Ok`, `USRHOOKS_Error`)

## Design Patterns & Rationale

**Hook/Adapter Pattern**: Rather than hard-coding malloc/free throughout audiolib, this module creates a controlled extension point. The host application can:
- Override these functions to use custom allocators (arena allocators, fixed pools)
- Track allocations for debugging or limits
- Integrate with application-level memory management (e.g., if ROTT uses a resource manager)

**Error-Driven API**: Returns status codes (`USRHOOKS_Ok` / `USRHOOKS_Error`) instead of relying on NULL checks or exceptions. This is idiomatic to 1990s C libraries and simplifies error propagation up the audio stack.

**Output Parameter Pattern**: `USRHOOKS_GetMem(void **ptr, size)` uses double-indirection rather than returning a pointer. This allows the function to return a status code while passing the allocated pointer out-of-band—common in older C codebases before better error handling emerged.

## Data Flow Through This File

```
Game/App calls audiolib function (e.g., BLASTER_Init)
  → audiolib code needs buffer (e.g., DMA ring buffer)
  → calls USRHOOKS_GetMem
    → malloc() from heap
    → [If success: store ptr, return USRHOOKS_Ok]
    → [If failure: return USRHOOKS_Error, caller must handle]
  → audiolib checks status, propagates error or uses buffer
  
On shutdown:
Game/App calls audiolib cleanup (e.g., BLASTER_Shutdown)
  → audiolib code calls USRHOOKS_FreeMem
    → free() returns memory to heap
    → returns USRHOOKS_Ok or USRHOOKS_Error if ptr was NULL
```

## Learning Notes

**Idiomatic 1990s Design**:
- No exceptions, assertions, or NULL-pointer guarantees
- Status codes for error reporting (precursor to modern Result/Option types)
- Explicit output parameters instead of return values
- Hardware alignment assumptions (dword) suggest target: x86 DOS/Windows era

**Modern Contrast**: Today's Rust/C++ would use:
- Exceptions or Result<T, E> for errors
- Returning pointers directly from allocators
- RAII for automatic cleanup (no manual `FreeMem` calls)

**Integration Insight**: This file demonstrates how a library can remain *portable and customizable* by never hard-coding system dependencies. The host program decides what malloc is—it could be:
- Standard C malloc (default implementation here)
- Custom arena allocator for fixed-size pools
- Memory-mapped file allocator for persistent audio state
- Bounds-checking wrapper for debugging

The fact that usrhooks.h is declared here but likely included by all audiolib internals reinforces this as a **critical integration seam** between the audio library and the broader engine.

## Potential Issues

- **No size validation**: `USRHOOKS_GetMem` does not check for integer overflow or excessive allocations; assumes the caller validates
- **Null-pointer assertion**: `USRHOOKS_FreeMem` treats NULL as an error rather than a no-op (which is safer but requires precise ownership tracking by the caller)
- **No alignment guarantee**: The comment states memory is "assumed to be dword aligned," but malloc on some platforms may not honor this without explicit requests (though x86 typically aligns to 8+ bytes)
