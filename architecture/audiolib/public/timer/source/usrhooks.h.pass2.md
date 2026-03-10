# audiolib/public/timer/source/usrhooks.h — Enhanced Analysis

## Architectural Role

This header defines the **memory management injection interface** for the audio library. It creates a contract between the audio library core (which includes device drivers, DSP managers, and MIDI handlers) and the calling application, allowing the app to intercept all dynamic allocations. This is critical for 1990s game engines running on memory-constrained DOS systems, where the game needs strict control over heap usage, fragmentation, and allocation tracking.

## Key Cross-References

### Incoming (who depends on this file)
- **Game engine code** (rott/* implies main application) must implement these two functions to initialize the audio subsystem
- **Audio device drivers** (blaster.c, al_midi.c, awe32.c visible in cross-ref) call these during initialization and runtime
- Declared in public API (under `audiolib/public/`) so all audio clients depend on this contract

### Outgoing (what this file depends on)
- No outgoing dependencies; this is pure interface definition
- Callers must provide implementations; the header itself is dependency-free

## Design Patterns & Rationale

**Dependency Injection via Function Pointers (Callbacks)**
- Instead of hardcoding `malloc`/`free`, the library calls application-provided functions
- Enables **memory pooling**: game can pre-allocate a pool and serve from it
- Enables **tracking & instrumentation**: game logs all allocations for debugging
- Enables **restricted heaps**: game can isolate audio memory from game memory
- Common in DOS-era code where every byte and allocation cycle mattered

**Why a double pointer for GetMem?**
- `void **ptr` is an output parameter—allows the hook to write the allocated address back
- Pattern: `USRHOOKS_GetMem(&my_pointer, size)` fills in `*my_pointer`
- Safer than returning a void* (return value reserved for error code)

## Data Flow Through This File

1. **Initialization**: Game calls audio library init function (e.g., `AL_Init`, `ADLIBFX_Init`)
2. **Hook registration** (implied): Audio library holds references to these functions, or game registers them globally
3. **Runtime allocation**: Audio library calls `USRHOOKS_GetMem(size)` when it needs buffers (DMA, voice data, MIDI patches)
4. **Cleanup**: Audio library calls `USRHOOKS_FreeMem(ptr)` when shutting down or releasing voices
5. **Control flow**: All memory operations flow through the game, not the library

## Learning Notes

**Idiomatic to era**: This is typical of 1990s commercial game libraries—they exposed hooks for memory, interrupts, timing, and I/O because embedded/real-time constraints meant libraries couldn't assume a standard runtime environment.

**Modern equivalent**: Today, this would be dependency injection via constructor, a custom allocator trait, or a `malloc_stats` callback. The principle is identical—library provides hooks, caller provides policy.

**Game engine lesson**: Multi-subsystem engines (audio, video, physics, AI) all need this pattern to coexist gracefully. Apogee's design separates **mechanism** (audio DSP handling) from **policy** (where memory comes from).

## Potential Issues

- **No validation**: The hooks receive raw sizes with no bounds checking—caller could request unlimited memory
- **No context**: Hooks don't know *why* the library needs memory (buffer, temporary, persistent), so the game can't apply differentiated policies
- **Error semantics unclear**: `USRHOOKS_Error` is returned on failure, but what does the library do if allocation fails? (Header doesn't specify)
- **No callback storage**: The header declares the interface but *doesn't show how the library obtains references to these functions*—implementation likely uses global function pointers or linker tricks
