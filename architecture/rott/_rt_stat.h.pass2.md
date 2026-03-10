# rott/_rt_stat.h — Enhanced Analysis

## Architectural Role

This private header is the internal API layer for ROTT's static object (doors, walls, light sources, decorations) subsystem. It sits between the object construction pipeline (level loading, save/load) and the public static management interface (`rt_stat.h`). The file defines internal registration functions and state serialization structures that support level initialization and game persistence.

## Key Cross-References

### Incoming (who depends on this file)
Based on cross-reference analysis:
- **Level loader** — calls `AddStatic()` and `AddAnimStatic()` during level construction to register objects into the game world
- **Initialization system** — calls `PreCacheStaticSounds()` at startup to buffer audio resources
- **Save/load routines** — use `saved_stat_type` to serialize and restore static object state in save games
- **Public API consumers** — the public header `rt_stat.h` re-exports `AddStatic` and likely wraps these private functions

### Outgoing (what this file depends on)
- `statobj_t` type — imported from elsewhere (likely `_rt_acto.h` or similar actor/object system)
- Global `sprites[x][y]` array — 2D spatial grid for light detection (likely managed by render/scene system)
- Sound caching subsystem — `PreCacheStaticSounds()` integrates with audio initialization pipeline
- No direct calls visible in this header; implementation (`rt_stat.c`) handles the actual work

## Design Patterns & Rationale

**Private Implementation Header Pattern**: This file uses the classic C split—`_rt_stat.h` for internal APIs, `rt_stat.h` for public. This isolates implementation complexity and prevents external code from depending on volatile internal functions.

**Type-Safe Object Registration**: `AddStatic()` and `AddAnimStatic()` take a `statobj_t*` pointer, allowing the static system to register any object type without knowing its internal structure (structural subtyping in C idiom).

**Lazy Audio Caching**: `PreCacheStaticSounds(int)` suggests sounds are grouped (the int likely identifies a group or count), enabling selective caching to reduce memory overhead—only load audio needed for the current level.

**Dual Storage for Statics**: `awallinfo_t` (runtime animation state) and `saved_stat_type` (serialized state) separate concerns—one optimized for gameplay, one for persistence.

## Data Flow Through This File

1. **Level Load Path**: 
   - Level file parsed → `AddStatic()` / `AddAnimStatic()` called per object → objects registered into spatial/actor systems
   - Parallel: `PreCacheStaticSounds()` called to buffer audio

2. **Save/Load Path**:
   - Game state → `saved_stat_type` snapshot (position, flags, hitpoints, animation counters, linked references)
   - Load: `saved_stat_type` → restore object state via game loop

3. **Runtime Animation**:
   - `awallinfo_t` drives wall animation frames via `AnimateWalls()` (in public header)
   - `IsLight()` macro checks `sprites[x][y]->flags & FL_LIGHT` for lighting queries

## Learning Notes

**1. Era-Specific Design**: This structure reflects early 1990s engine architecture—no dynamic systems (ECS), instead static registration during load and explicit state snapshots. Modern engines use prefabs or data-driven spawning.

**2. Fixed Spatial Grid**: The `sprites[x][y]` global (128×128 cells implied by `IsLight` bounds) is a baked-in acceleration structure for light detection. This trades memory for O(1) light queries—very different from modern octree/BVH approaches.

**3. Animation as Metadata**: `awallinfo_t` suggests animations are declarative (timing info, lump references) rather than procedural, fitting the "data-driven asset" philosophy of id Tech 1 engines (Doom, Heretic).

**4. Linked Objects**: `saved_stat_type.linked_to` field hints at chains of dependent objects (e.g., a door triggering decorations). This is a simple graph structure, not a full node system.

## Potential Issues

**Type Safety Gap**: `saved_stat_type` is a generic container for any static object, but there's no type tag or discriminator. If different statics have different fields, deserialization could corrupt state (offset misalignment).

**Implicit Coupling**: `IsLight()` directly accesses the global `sprites` array without bounds validation on the input (x, y). If called with out-of-range coordinates in gameplay code, it would silently return false or cause undefined behavior.

**Animation Timing Mismatch**: `awallinfo_t.tictime` and `saved_stat_type.ticcount` suggest animation state is split across runtime and saved structures—risk of desync if both aren't updated consistently during save.
