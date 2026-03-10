# audiolib/source/usrhooks.h — Enhanced Analysis

## Architectural Role

This file defines the **memory allocation/deallocation hook interface** that decouples the audio library from the host application's memory management. All audio subsystems (BLASTER, ADLIBFX, AL_MIDI, AWE32, GUS) that need memory during init/shutdown call through these hooks instead of directly using malloc/free. This allows the calling application (ROTT engine) to enforce memory budgets, use custom allocators, or operate in restricted memory environments.

## Key Cross-References

### Incoming (who depends on this file)

Based on codebase architecture, these audio driver modules call into these hooks:
- **audiolib/source/blaster.c** — BLASTER_Init, BLASTER_Shutdown (DMA buffer allocation)
- **audiolib/source/adlibfx.c** — ADLIBFX_Init, ADLIBFX_Shutdown 
- **audiolib/source/al_midi.c** — AL_Init, AL_Shutdown (voice/timbre management)
- **audiolib/source/awe32.c** — AWE32_Init, AWE32_Shutdown
- **audiolib/source/gus.c** — GUS module initialization
- **rott/rt_main.c** or equivalent — Must implement both functions; called during engine startup/shutdown

### Outgoing (what this file depends on)

- None (pure interface definition; no dependencies on other headers)
- Implicitly requires: caller's implementation of USRHOOKS_GetMem and USRHOOKS_FreeMem

## Design Patterns & Rationale

**Hook/Callback Pattern**: Rather than hardcoding malloc/free, the library defines a contract (this header) that the caller must fulfill. The caller retains control over memory allocation strategy.

**Dependency Inversion**: Audio drivers depend on an abstraction (these hooks) rather than on the C standard library. This allows:
- Memory pooling or slab allocation in the game engine
- Tracking allocations for debugging
- Strict memory quotas (e.g., "audio system gets max 2MB")
- Custom error handling (returning failure instead of crashing on OOM)

**Tradeoff**: Two-level indirection (caller implements, driver calls) vs. simplicity of malloc/free. Necessary in 1994–1995 embedded/DOS era where memory was scarce and strict control was critical.

## Data Flow Through This File

```
Engine Initialization
  ↓
BLASTER_Init, AL_Init, ADLIBFX_Init, ...
  ↓
USRHOOKS_GetMem(ptr, size)  ← calls into engine-provided function
  ↓
Engine allocates from its heap/pool, returns pointer
  ↓
Audio driver uses allocated buffer (DMA, voice data, timbre banks)
  ↓
Engine Shutdown
  ↓
USRHOOKS_FreeMem(ptr)  ← calls into engine-provided function
  ↓
Engine releases memory
```

## Learning Notes

**Idiomatic 1990s game engine pattern**: Modern engines use memory arenas, custom allocators, and ECS-style resource pools. This hook pattern is the pre-ECS way of achieving separation of concerns—the library doesn't own memory; the engine does.

**Contract-based programming**: The enum `USRHOOKS_Errors` with three states (Ok, Error, Warning) signals that allocation can fail gracefully. Modern engines might throw exceptions; this era required explicit error codes and caller checks.

**Why double-pointer for output?** `USRHOOKS_GetMem(void **ptr, ...)` — this pattern (output parameter via double-pointer) avoids returning both a pointer and error code; the return value is solely for error reporting. Common in C APIs before out-of-band error mechanisms.

## Potential Issues

- **No bounds checking**: Caller's implementation must validate `size` (e.g., no integer overflows). The header doesn't specify max allocation size or alignment requirements.
- **Dangling pointers**: If audio driver frees memory twice or frees unallocated pointers, the caller's USRHOOKS_FreeMem must handle gracefully (no crash).
- **Ownership ambiguity**: Caller must ensure that USRHOOKS_FreeMem is only called on pointers returned by USRHOOKS_GetMem, in matching pairs. No mechanism to verify this—enforced by protocol only.
