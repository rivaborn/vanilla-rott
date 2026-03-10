# rott/w_wad.h — Enhanced Analysis

## Architectural Role

`w_wad.h` defines the resource loading backbone of the ROTT engine. Every lump (sprite, texture, map, audio metadata, etc.) flows through this interface. The WAD system bridges persistent disk storage and runtime memory, operating as a cache layer with lifetime management via allocation tags. Since the engine is completely asset-driven—even map data, actor sprites, and UI graphics live in lumps—this is a foundational subsystem that all gameplay, rendering, and audio systems depend on.

## Key Cross-References

### Incoming (who depends on this file)
**Likely heavy consumers** (inferred from typical engine architecture; full cross-ref excerpt incomplete):
- **Rendering system** (`rt_draw.c`, `rt_view.c`) — fetches sprite/texture lumps
- **Map system** (`rt_door.c`, `rt_map.c`) — loads map data lumps
- **Game state** (`rt_game.c`) — loads savefiles and config data stored as lumps
- **Actor/sprite system** (`rt_actor.c`) — caches actor sprite lumps by name
- **Menu system** (`rt_menu.c`) — loads UI graphics, fonts, palette lumps
- **Engine init** (`rt_main.c`, `engine.c`) — calls `W_InitMultipleFiles` during startup

### Outgoing (what this file depends on)
- **Memory manager** (inferred from `int tag` parameter in cache functions) — likely `Z_*` memory system with lifetime tracking
- **Disk I/O** (implicit via `W_InitFile`/`W_InitMultipleFiles`) — reads WAD binary files from disk during init
- **C standard library** — string operations for lump name lookups

## Design Patterns & Rationale

**1. Lazy Caching with Tags**  
The `W_CacheLumpNum/Name` functions accept a `tag` parameter, integrating with a garbage-collected memory pool. This allows:
- Automatic eviction when tag scope ends (level transition, menu exit)
- No explicit deallocation burden on callers
- Memory reuse without fragmentation

This is classic **Doom-era** design (ROTT's spiritual predecessor).

**2. Name → Index Indirection**  
`W_GetNumForName` / `W_CheckNumForName` provide semantic lookups while maintaining a flat lump array. This decouples callers from lump order and allows:
- Human-readable asset names in code
- Flexible WAD composition (lumps can be in any order)
- Fast numeric indexing after lookup

**3. Immutable Lump Registry**  
`numlumps` and `lumpcache` are globals populated once at init. No runtime lump addition—assets are fixed after `W_InitFile` calls. This is safe, cache-friendly, and typical of 1990s asset pipelines where all content was baked during build.

## Data Flow Through This File

**Startup Phase:**
```
engine.c: main()
  ↓ (calls W_InitMultipleFiles with list of .wad filenames)
  ↓
W_InitMultipleFiles: Reads each WAD file, populates numlumps, creates lump directory
  ↓ (by reference, for each lump in WAD)
  ↓
lumpcache[]: initialized to NULL (lumps not loaded yet)
```

**Runtime Phase:**
```
rt_draw.c: (rendering sprite for actor)
  ↓ (calls W_CacheLumpName("ACTOR_SPRITE_01", tag=SPRITE_TAG))
  ↓
W_CacheLumpName: Calls W_GetNumForName → looks up lump index
  ↓
W_CacheLumpNum: Checks lumpcache[index]; if NULL, W_ReadLump into fresh allocation
  ↓ (updates lumpcache[index])
  ↓
Returns void* → caller casts to sprite_t* and uses
  ↓ (when tag scope ends, memory manager frees via tag)
```

## Learning Notes

**Idiomatic to 1990s Engines:**
- WAD file format (borrowed from Doom, itself from Wolfenstein 3D)—a simple concatenated archive with a directory appended at end
- Global lump registry—pre-loaded, read-only after init. Modern engines use asset managers with hot-reloading and streaming.
- C-style memory tags—precursor to RAII/scoped allocators. Memory tied to a simple integer lifetime scope rather than language semantics.

**Engine Architecture Insight:**
This file reveals ROTT is **not** streaming-based. All lumps referenced in a level must fit in RAM or be swapped manually. No virtual memory or async I/O abstractions visible here.

**Name Lookup Overhead:**
`W_GetNumForName` likely does a linear string search through all lumps. For a 1990s engine with hundreds, not tens of thousands, of lumps, this is acceptable but would be a bottleneck in modern engines (use hashing).

## Potential Issues

**No obvious bugs, but architectural constraints:**
- **Name collision risk**: If two WAD files define lumps with the same name, behavior depends on load order and isn't specified here. Modern engines namespace or error on collision.
- **Cache invalidation**: Once a lump is cached, there's no explicit purge mechanism visible. If a lump must be reloaded, unclear if calling `W_CacheLumpNum` again returns stale data or fresh data.
- **No error reporting**: Functions like `W_GetNumForName` don't clarify what happens if a lump isn't found—likely returns 0 or -1; callers must handle gracefully.
