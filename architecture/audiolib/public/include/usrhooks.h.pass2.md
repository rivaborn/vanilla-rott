# audiolib/public/include/usrhooks.h — Enhanced Analysis

## Architectural Role

This header is the **memory abstraction layer** for the entire audio library ecosystem. Every audio subsystem (Sound Blaster, AdLib, GUS, AWE32, MIDI) that needs dynamic allocation delegates through these two function pointers, allowing the game engine to enforce its own memory budgets, fragmentation policies, and allocation tracking. This is a foundational constraint that permeates all audio initialization and resource management.

## Key Cross-References

### Incoming (who depends on this)
- **All audio subsystem implementations**: BLASTER (blaster.c), ADLIBFX (adlibfx.c), AL_MIDI (al_midi.c), AWE32 (awe32.c), GUS driver family
- **Audio initialization chain**: Any `*_Init()` call in the audio library (BLASTER_Init, AL_Init, AWE32_Init) will call `USRHOOKS_GetMem` during setup
- **Game engine (rott)**: The calling application must provide implementations of these functions before initializing any audio subsystem

### Outgoing (what this defines)
- **No external dependencies** — this header only declares the plugin interface
- **Implementations live in**: USRHOOKS.C (in game code), which the game provides
- The audio library reads no globals; it only calls the provided function pointers

## Design Patterns & Rationale

**Strategy Pattern (via function pointers)**: The library doesn't hardcode malloc/free. Instead, it accepts a strategy at link time.

**Why this design**:
- 1994–95 era constraint: DOS/early Windows had fragmented memory, varied allocators (conventional, extended, etc.)
- Game needs control: Can enforce per-subsystem budgets, pre-allocate pools, or swap allocators based on hardware
- Clean decoupling: Audio library has zero dependency on libc malloc

**Tradeoff**: Caller *must* provide both functions before any audio init. Failure is silent if not provided — no safety net (see Potential Issues).

## Data Flow Through This File

1. **Game startup**: Game implements `USRHOOKS_GetMem` / `USRHOOKS_FreeMem`
2. **Audio subsystem init** (e.g., BLASTER_Init):
   - Calls `USRHOOKS_GetMem(ptr, size)` to allocate voice buffers, DMA structures, etc.
   - On success (`USRHOOKS_Ok`), subsystem owns the pointer and proceeds
   - On failure, init aborts (error propagates up)
3. **Runtime**: Audio plays; no further allocations typically needed
4. **Shutdown**: Subsystem calls `USRHOOKS_FreeMem` to release all allocated blocks

The return codes allow graceful degradation: if memory is tight, a subsystem could partially degrade or skip optional features rather than crash.

## Learning Notes

**Idiomatic to this era**:
- Explicit, caller-provided allocators were standard (Duke3D, Doom also do this)
- No RAII, destructors, or exception safety; caller tracks ownership
- Double-indirection on `void **ptr` is typical for "output parameter" patterns in C

**Modern equivalent**: Dependency injection with allocator traits (C++) or custom allocator strategies (Rust).

**Game engine integration**: This shows how a 3rd-party audio library achieves "hostility" in a closed environment — the game controls all resources, not the library.

## Potential Issues

- **No validation**: If the game doesn't provide these functions (linker error aside), calling code has undefined behavior. The interface assumes cooperation, not defense.
- **No error recovery**: If `USRHOOKS_GetMem` fails mid-stream (e.g., during play), there's no standardized panic mechanism — depends on caller's error handling.
- **Pointer lifetime**: Library trusts the game won't free memory it's still using. No reference counting or allocation tracking visible in this header.
