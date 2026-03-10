# audiolib/public/ps/usrhooks.h — Enhanced Analysis

## Architectural Role
This file defines the memory abstraction boundary between the audio library and its calling application. It enables the **audio subsystem to remain agnostic of memory management policy**—the game engine controls allocation strategy (pooling, tracking, limits) while the audio library simply requests and releases blocks. This inverts the dependency: audio code depends on abstractions (the hook interface), not on concrete memory implementations.

## Key Cross-References
### Incoming (who depends on this file)
- **All audio subsystems** in `audiolib/source/` that allocate memory call these hooks:
  - BLASTER (Sound Blaster) functions (`blaster.c`)
  - AL_* (MIDI/synth) functions (`al_midi.c`)
  - AWE32 (AWE32 synth) functions (`awe32.c`)
  - ADLIBFX (AdLib FM) functions (`adlibfx.c`)
  - GUS (Gravis UltraSound) functions (`gus.c`)
  - Any runtime memory needs in these drivers flow through `USRHOOKS_GetMem`/`USRHOOKS_FreeMem`

### Outgoing (what this file depends on)
- None directly; this is a pure interface definition
- **Implemented by**: The calling application (likely `rott/` game engine or initialization code in `audiolib/public/ps/ps.c` or `audiolib/public/pm/source/pm.c`)
- The game engine must provide concrete implementations before calling any audio init functions

## Design Patterns & Rationale
- **Hook/Callback Pattern**: Rather than audio code calling `malloc`/`free` directly, it calls standardized hook functions. This decouples allocation policy from audio logic.
- **Dependency Inversion**: Audio library depends on the abstraction (USRHOOKS interface), not on the concrete memory manager.
- **Strategy Pattern**: The calling program can swap allocation strategies (linear allocator, pool, tracking wrapper, etc.) without recompiling the audio library.
- **Why this design?** (Inferred from 1994 era context):
  - DOS/early Windows had fragmented memory, pools, UMBs (upper memory blocks)—fixed strategies failed
  - Game needed to control memory layout to prevent fragmentation
  - Modular library design required dependency injection before dynamic linking existed

## Data Flow Through This File
1. **Initialization phase**: Game engine registers hook implementations (sets function pointers in audio driver init)
2. **Runtime allocation**: Audio driver calls `USRHOOKS_GetMem(void **ptr, size)`
   - Game's implementation allocates from its pool/arena
   - Writes pointer to `*ptr`
   - Returns `USRHOOKS_Ok` (0) or error code
3. **Runtime deallocation**: Audio driver calls `USRHOOKS_FreeMem(ptr)`
   - Game's implementation marks block free or returns to pool
   - Returns status

**Key assumption**: The audio library trusts the application to implement these correctly; it doesn't validate pointer validity or track allocations itself.

## Learning Notes
- **1994-era memory discipline**: Explicit allocation/deallocation with error codes, no exceptions, no RAII
- **Contrast with modern engines**:
  - Modern: Custom allocators, bump allocators, memory arenas built into the language/runtime
  - 1994: Manual allocation hooks, no garbage collection, careful tracking to avoid leaks
- **Idiomatic to this engine**: Shows modular library design philosophy—audio library is a self-contained subsystem that assumes nothing about the caller's runtime
- **Game engine integration point**: This header is the contract the game engine **must satisfy** to use the audio library; failure to implement these correctly (double-free, buffer overrun in allocated memory) will corrupt the audio driver state

## Potential Issues
- **Double-free risk**: If the game engine's implementation of `USRHOOKS_FreeMem` frees a pointer twice, or the audio library calls it with an invalid pointer, memory corruption occurs with no bounds checking
- **Leak risk**: If audio library calls `USRHOOKS_GetMem` but never calls the corresponding `USRHOOKS_FreeMem`, or allocates then crashes, the game engine's pool may leak blocks
- **No validation**: The hook interface passes bare pointers with no type information; the audio library trusts the implementation to allocate the correct size and return a valid address
- **Silent failures**: A hook implementation that returns `USRHOOKS_Error` is handled by the caller; if not checked, audio code may proceed with a null or invalid pointer

---

**Sources for cross-reference mapping**: Audio subsystems (BLASTER, AL_MIDI, AWE32, ADLIBFX, GUS) all visible in cross-reference index; their `Init` functions would register these hooks.
