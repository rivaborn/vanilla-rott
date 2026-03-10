# rott/cin_actr.c — Enhanced Analysis

## Architectural Role

`cin_actr.c` provides the **lightweight container layer** for cinematic effects within the broader cinematic subsystem. It maintains a global registry of active effects and orchestrates their per-frame lifecycle (update, render) without implementing effect logic itself. This file acts as a **rendering scheduler and actor manager** that bridges the cinematic main loop (likely in `cin_main.c`) with effect-specific handlers in `cin_efct.c`, following a **data-driven, effect-type-agnostic pattern** where new cinematic effects can be added without modifying this core infrastructure.

## Key Cross-References

### Incoming (who depends on this file)
- **cin_main.c** (`CacheScriptFile`): The cinematic script loader/executor likely calls `SpawnCinematicActor()` when parsing script events
- **cin_evnt.c** (`CreateEvent`, `AddEvent`): Event creation system probably spawns actors via `SpawnCinematicActor()` for script-driven effects
- **Game main loop** (rt_game.c/rt_main.c): Likely calls `StartupCinematicActors()` at cinematic start, then per-frame calls to `UpdateCinematicActors()` and `DrawCinematicActors()`
- Public header `cin_actr.h` exports: `AddCinematicActor`, `firstcinematicactor`, `lastcinematicactor` globals (visible to cin_efct.c for direct list manipulation if needed)

### Outgoing (what this file depends on)
- **cin_efct.c**: `UpdateCinematicEffect(type, effect_ptr)` and `DrawCinematicEffect(type, effect_ptr)` — effect-specific update/render dispatch (likely a switch or function table)
- **modexlib.c**: `XFlipPage()` — video buffer flip at end of each render phase
- **memcheck.h**: `SafeMalloc()`, `SafeFree()` — memory allocation with tracking/safety
- **cin_def.h**: `enum_eventtype` (13+ effect types), `actortype` struct definition, `MAXCINEMATICACTORS` constant
- **cin_glob.h**: Global cinematic state (possibly abort flags, delay counters)
- **Standard library**: `Error()` for overflow handling

## Design Patterns & Rationale

### 1. **Lightweight Container Pattern**
Actors are inert data containers (type + opaque pointer), not active objects. Actual behavior is delegated to effect-type-specific handlers in `cin_efct.c`. This **decouples actor lifecycle from effect logic**, allowing:
- New effect types to be added without modifying `cin_actr.c`
- Shared allocation/deallocation infrastructure across heterogeneous effects
- Simple per-frame iteration without polymorphism overhead

### 2. **Doubly-Linked List with Tail Pointer**
- **Why**: `AddCinematicActor()` achieves O(1) append (common case: spawning effects mid-sequence)
- **Trade-off**: Deletion is still O(1) with saved pointers, but iteration is linear. For small max count (MAXCINEMATICACTORS = 30), this is negligible
- **Tradeoff vs. array**: Array would be O(n) append or waste space; linked list allows unbounded growth up to max

### 3. **Phased Rendering System**
`DrawCinematicActors()` iterates **7 phases** in fixed order:
1. Screen functions (fade, blank, clear)
2. Background
3. Background sprites
4. Backdrop
5. Foreground sprites
6. Palette functions
7. Page flip timing

**Rationale**: Ensures visual layering without z-depth: backgrounds first, then sprites, ensuring correct occlusion. Screen functions and palette ops prevent intermediate page flips (`flippage=false`) to batch updates.

### 4. **Safe Iteration During Modification**
Both `UpdateCinematicActors()` and `DrawCinematicActors()` save `nextactor` before calling deletion—**preventing iterator invalidation** when effects complete mid-frame. This is a classic C pattern for safe removal during iteration.

### 5. **Singleton Initialization Pattern**
`StartupCinematicActors()` with `cinematicactorsystemstarted` guard is **idempotent**—safe to call multiple times. Mirrors game init/shutdown lifecycle.

## Data Flow Through This File

```
Script/Event System (cin_main.c, cin_evnt.c)
    ↓ SpawnCinematicActor(type, effect_ptr)
cin_actr.c: GetNewCinematicActor() → AddCinematicActor()
    ↓ (linked list of active actors)
Game Main Loop (per frame):
    ├─ UpdateCinematicActors()
    │   └─ UpdateCinematicEffect(actor->type, actor->effect) [cin_efct.c]
    │       ↓ returns false if complete
    │       └─ DeleteCinematicActor() [removed from list, freed]
    │
    └─ DrawCinematicActors() [7-phase render loop]
        └─ for each phase, for each actor:
            ├─ DrawCinematicEffect(actor->type, actor->effect) [cin_efct.c]
            │   ↓ calls XFlipPage() (modexlib.c)
            └─ if returns false → DeleteCinematicActor()

Shutdown (cin_main.c):
    └─ ShutdownCinematicActors() → DeleteCinematicActor() all → free all
```

**Key state transitions:**
- Actor created → added to list → updated/drawn each frame → marked complete (effect returns false) → deleted and freed

## Learning Notes

### Idiomatic 1990s C Game Engine Patterns
1. **Explicit linked lists** instead of vectors/dynamic arrays (common in retro C engines)
2. **Opaque void pointers** for polymorphism instead of inheritance or callbacks (lightweight, but requires type discipline)
3. **Global state management** with file-static guards (`cinematicactorsystemstarted`) for "systems"
4. **Phased rendering** as an alternative to z-ordering (more explicit, better for batch updates in fixed-function graphics pipelines)
5. **Per-frame lifecycle hooks** (update, draw) integrated into a main loop, not event-driven

### Modern Engines Would Do Differently
- **ECS or component-based**: Actors would be entities with components; systems would iterate components, not actor types
- **Deferred rendering**: Separate command buffers per phase instead of runtime branching
- **Virtual functions or function tables**: Avoid the cin_efct.c dispatch entirely with polymorphism
- **Double-buffered lists**: Avoid iterator invalidation with separate "to_delete" lists

### Connections to Engine Concepts
- **Actor pool**: Preallocated up to MAXCINEMATICACTORS (pool pattern, though not explicit object reuse here)
- **Scene graph layer**: The 7 render phases approximate a simplified scene graph depth ordering
- **Separation of concerns**: Actor container (this file) vs. effect implementation (cin_efct.c) mirrors MVC

## Potential Issues

1. **No handle/ID system**: Direct pointers to actors are stored by caller. If caller loses pointer and tries to spawn new actors, dangling references could occur. Safer approach: return opaque IDs and maintain a handle table.

2. **Unbounded effect data**: `actor->effect` is an opaque void pointer. No size tracking means `DeleteCinematicActor()` calls `SafeFree(actor->effect)` blindly. If effect data is a static struct or embedded, double-free could crash.

3. **Debug code (DUMP flag)**: Conditional `printf()` statements remain in release code, suggesting this was instrumented during development but not cleaned up.

4. **Insufficient comment on ownership**: Caller must allocate effect data and pass pointer. On deletion, `SafeFree(actor->effect)` is called—but if effect data is shared or stack-allocated elsewhere, this causes undefined behavior. First-pass notes this is unclear.

5. **No error handling for effect handlers**: `UpdateCinematicEffect()` and `DrawCinematicEffect()` can fail (both return boolean), but error codes are not propagated. Calling code has no way to know *why* an effect failed (logic complete vs. resource error).
