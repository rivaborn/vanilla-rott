# rott/usrhooks.c — Enhanced Analysis

## Architectural Role

This file bridges the **audiolib library** (and other bundled subsystems) with ROTT's internal **Z_Zone memory manager**. Rather than letting libraries use raw `malloc/free`, all library memory requests funnel through these hooks, enabling centralized allocation policy and tracking. The explicit public hook design signals that game customizers could redirect this to alternate allocators (e.g., fixed pools, debug heaps) without touching library code.

## Key Cross-References

### Incoming (who depends on this file)

- **audiolib/** (all audio subsystems: BLASTER, ADLIBFX, AL_MIDI, AWE32, GUS, etc.) — calls `USRHOOKS_GetMem`/`USRHOOKS_FreeMem` for initialization and voice/buffer management
- Likely called from **library initialization code** during engine startup (audio hardware detection, MIDI setup, DMA buffers)
- No direct calls observed from game logic (rt_*.c), consistent with "library hook" design

### Outgoing (what this file depends on)

- **z_zone.h** → `Z_Malloc(size, PU_STATIC, NULL)` — allocates from the engine's zone heap with static-lifetime tag
- **z_zone.h** → `Z_Free(ptr)` — returns memory to zone manager without explicit user tag
- **memcheck.h** — optional debug instrumentation (compiles out if `NOMEMCHECK` undefined)

## Design Patterns & Rationale

1. **Library Adapter / Facade Pattern**  
   Decouples library memory allocation from engine memory policy. Libraries ask for memory through a known interface; engine retains control over *where* that memory comes from.

2. **Error-Status Return Pattern (vs. Exceptions)**  
   Returns `int` codes instead of exceptions—consistent with 1990s C idioms and the audiolib's pre-exception era. Caller must check return values (though many likely don't, per USRHOOKS_Ok = 0 = falsy in C).

3. **Output Parameter for Pointer**  
   `USRHOOKS_GetMem(void **ptr, ...)` uses an output parameter rather than returning the pointer directly, matching the audiolib's expected signature and allowing in-place allocation into caller's data structures.

4. **Asymmetric Error Handling**  
   `GetMem` validates Z_Malloc's NULL return; `FreeMem` assumes Z_Free succeeds and never reports failure. This suggests Z_Free is trusted to always succeed (likely fatal if it fails anyway).

## Data Flow Through This File

**Initialization Phase:**
1. Audio hardware detection / library init calls `USRHOOKS_GetMem(buffer, size)`
2. Gets routed to `Z_Malloc(size, PU_STATIC, NULL)` — allocates static-lifetime zone memory
3. Success/failure reported back; library stores pointer
4. Memory persists until shutdown (PU_STATIC = never tagged for purge)

**Shutdown Phase:**
1. Library calls `USRHOOKS_FreeMem(buffer)`
2. Routed to `Z_Free(ptr)`; memory returned to zone
3. No explicit error feedback (caller trusts it works)

## Learning Notes

**1940s–1990s Game Engine Idioms**
- Explicit output parameters instead of return values (vs. modern Rust/Go multiple returns)
- No exception-based error handling; explicit status codes
- Memory tied to fixed lifetime tags (PU_STATIC vs. PU_LEVEL) rather than RAII/destructors

**How Modern Engines Differ**
- Would use C++ constructors/destructors or Rust lifetimes for automatic cleanup
- Would likely use a single allocator interface, not adapter layers for libraries
- Memory would be scope-tracked, not lifetime-tagged

**Idiomatic to ROTT**
- The PU_STATIC tag reveals ROTT uses a **zone-based memory system** with explicit lifetime management—a hallmark of Doom-engine lineage
- The public hook design shows ROTT was built with **modding/customization in mind** (unlike closed-box id Tech engines)
- Separating library allocation from game allocation enables **audit trails** and **quota enforcement** per subsystem

**Connections to Engine Concepts**
- This is essentially a **memory allocation strategy pattern**—libraries depend on an abstract allocator, not concrete malloc
- Relates to **dependency injection**: the library is "injected" with a custom allocator
- Pre-dates DI frameworks; the code simply expects libraries to call known hooks

## Potential Issues

1. **Silent Allocation Failure Risk**: While `USRHOOKS_GetMem` returns error status, audiolib callers may not check it properly. If an audio buffer allocation fails quietly, subsystems might proceed with null pointers.

2. **Asymmetric Validation**: Only `GetMem` validates NULL; `FreeMem` silently accepts NULL but returns error. Defensive design would validate both directions consistently (though NULL-on-free may be intentional to match C standard library behavior).

3. **No Quota/Tracking**: The zone manager (Z_Malloc) handles allocation, but this file doesn't add any per-library quotas or logging—useful if diagnosing why audio init fails.

4. **PU_STATIC Lifetime Assumption**: All library allocations use PU_STATIC (persist entire game). If a library expects to free memory mid-level, this assumption breaks silently. (Unlikely for 1990s audiolib, but worth noting.)
