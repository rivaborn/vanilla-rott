# rott/rt_stat.c — Enhanced Analysis

## Architectural Role

**rt_stat.c** is the dynamic object registry and animation layer for Rise of the Triad's static environment. It operates as a parallel subsystem to **rt_actor.c** (enemy/player control), handling the spawn/removal lifecycle of 91 item and hazard types, lighting updates, and per-frame animation of both sprites and wall textures. Every explorable level is populated by calls to `SpawnStatic()` during level initialization (via the Ted map editor), making this file the foundation for game content placement and environmental interactivity.

## Key Cross-References

### Incoming (who depends on this file)
- **Level initialization** (`rt_ted.h`, Ted map parser): Calls `SpawnStatic()` for each static entity in the map
- **Respawn system** (internal): `CheckCriticalStatics()` is called per frame to tick respawn timers and re-spawn items
- **Player interaction** (`rt_playr.c`): Calls `ActivateLight()` / `DeactivateLight()` when light switches are toggled; interacts with respawn counters
- **Rendering** (`rt_draw.c`): Reads sprite spatial index `sprites[][]` for drawing; reads wall animations
- **Game logic** (`rt_main.c`, `rt_game.c`): Calls `DoSprites()`, `AnimateWalls()`, `CheckCriticalStatics()` each frame; calls `SaveStatics()` / `LoadStatics()` for persistence
- **Audio precaching** (menu/level load): Pre-caches sprite graphics and sound effects for faster playback

### Outgoing (what this file depends on)
- **Memory**: `z_zone.h` for level-permanent allocation (`Z_LevelMalloc`, `Z_Free`)
- **Graphics**: `rt_draw.h` (`SetLight`, `PreCacheLump`, `PreCacheGroup`); `lumpy.h` for sprite/patch data
- **Audio**: `rt_sound.h` for sound precaching and playback (`SD_PreCacheSound`, `SD_PlaySoundRTP`)
- **Map/world**: `rt_ted.h`, global `tilemap`, `MAPSPOT()`, `LightsInArea[]` for tile-based light propagation
- **Actor spawning**: `rt_actor.h`, `rt_main.h` (`GetNewActor()`, `SpawnNewObj()`) to spawn particles and linked objects
- **Utilities**: `rt_util.h` for tile-based lookup (`FindEmptyTile()`, `IsPlatform()`, `PlatformHeight()`) and random numbers
- **Game state**: `rt_main.h` (`gamestate`, `MISCVARS`), `rt_menu.h` (battle mode options), `rt_net.h` (multiplayer weapon persistence)

## Design Patterns & Rationale

**Object Pool + Doubly-Linked List (Active/Inactive/Free)**  
Rather than malloc/free per spawn, rt_stat.c pre-allocates a fixed pool and maintains three linked lists:
- `FIRSTSTAT/LASTSTAT`: Master list of all spawned statics (for serialization, cleanup)
- `firstactivestat/lastactivestat`: Animated statics only (culled from per-frame `DoSprites()` updates)
- `firstemptystat/lastemptystat`: Freed instances awaiting reuse

This avoids heap fragmentation (critical on early-90s DOS) and enables O(1) list removal via pointer fixup.

**Type Registry with Metadata**  
The `stats[NUMSTATS]` array decouples item definitions from code. Each entry stores sprite, flags, animation time, hitpoints, and ammo—allowing content to be added without recompilation. Flags (`FL_LIGHT`, `FL_WEAPON`, `FL_RESPAWN`) encode behavior polymorphically.

**Spatial Indexing**  
`sprites[MAPSIZE][MAPSIZE]` maps tile coordinates directly to sprite pointers, enabling O(1) collision checks and proximity queries (e.g., "is a light at this tile?").

**Deferred Respawn State Machine**  
`respawn_t` queue stores only essential state (position, type, spawn height); countdown happens each frame in `CheckCriticalStatics()`. This defers spawning until a safe tick, avoiding placement in invalid/occupied tiles.

**Lighting Propagation as Metadata Update**  
`TurnOnLight()` doesn't update graphics directly; it increments `LightsInArea[]` (used by pathfinding) and calls `SetLight()` (graphics backend). Separates semantic light state from rendering.

## Data Flow Through This File

1. **Level Load**:
   - Ted parser → `SpawnStatic(tilex, tiley, type, zoffset)` for each entity
   - Stat allocated (from free pool or `Z_LevelMalloc`)
   - Inserted into `FIRSTSTAT/LASTSTAT`, `sprites[][]`, optionally `firstactivestat` (if animated)
   - Sprite/sound precached to avoid hitches

2. **Per-Frame Update**:
   - `DoSprites()` iterates `firstactivestat`, increments ticcount, updates frame (shapenum)
   - `AnimateWalls()` updates wall animation frames (independent of sprites)
   - `CheckCriticalStatics()` decrements respawn timers; on expiry, calls `SpawnStatic()` again

3. **Player Interaction**:
   - Switch/lever hit → `ActivateLight()` → increments sprite frame, calls `TurnOnLight()` → updates light map
   - Light switch state persists in `switches[]` array

4. **Removal**:
   - `RemoveStatic()` unlinks from master list, optionally creates `respawn_t` entry
   - Stat moved to free pool
   - Weapon counter decremented (for battle mode)
   - Deferred respawn if `FL_RESPAWN` set and respawn enabled

5. **Save/Load**:
   - `SaveStatics()` serializes all statics + respawn queue to buffer (respawn indices resolved)
   - `LoadStatics()` reconstructs statics, rebuilds `sprites[][]` spatial index

## Learning Notes

- **Object Pool Design**: Typical 1990s pattern pre-dating modern allocators; demonstrates memory-conscious design for platform constraints.
- **Tile-Centric World**: Unlike modern engines with continuous coordinates, ROTT uses discrete tiles as the fundamental unit. Spatial index is a 2D grid, not a spatial hash.
- **Light Propagation Decoupled from Rendering**: `LightsInArea[]` serves pathfinding (actor AI avoids dark zones), while `SetLight()` updates the visual lightmap. Two semantic layers.
- **State Preservation via Metadata**: The `stats[]` registry makes save/load trivial—only spawn type and position needed, not full object state. Idiomatic for content-heavy games.
- **Respawn as Delayed Spawn**: Rather than instantly respawning, items tick down, then re-create themselves. Avoids edge cases (e.g., player standing on item's spawn tile) and allows SFX/particles to play.
- **Contrasts with Modern ECS**: No component composition here—behavior baked into flags and type-specific code. Monolithic `statobj_t` struct. Modern engines would decompose into health, drawable, animated systems.

## Potential Issues

1. **Spatial Index Not Cleaned on Remove**: When `RemoveStatic()` is called, `sprites[][]` is not explicitly cleared. If the stat is re-added later, a stale pointer may persist. (Verify in `AddStatic()` if it checks for collisions.)

2. **Respawn List Leaks if Level Exits**: `firstrespawn/lastrespawn` queue is rebuilt on level load (`InitStaticList()`), but if the player exits without dequeuing, memory may be lost. Verify cleanup in level shutdown.

3. **Light Animation Hard-Coded Directions**: `TurnOnLight()` has 8 corner-case branches; if a wall geometry change breaks assumptions, light propagation breaks without obvious error.

4. **Battle Mode Weapon Saturation**: `RemoveStatic()` checks `gamestate.BattleOptions.WeaponPersistence` to decide respawn, but weapon count (`MISCVARS->NUMWEAPONS`) is global. Race condition possible if two players pick up weapons simultaneously in netplay.

---

**Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>**
