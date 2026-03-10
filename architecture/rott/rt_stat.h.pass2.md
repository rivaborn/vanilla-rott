# rott/rt_stat.h — Enhanced Analysis

## Architectural Role

This header defines the runtime interface for static (non-actor) entities in the game world—the pooled object system that underlies environmental props, collectibles, hazards, and visual decorations. It acts as a bridge between the **level initialization subsystem** (which spawns objects from map data), the **main game loop** (which drives animation and lifecycle), and **rendering/physics subsystems** (which read positional and visual state). The static object system is fundamental to the engine's separation of concerns: actors (enemies, player) and statics (items, decorations, switches) are managed independently, allowing the renderer and physics code to operate on either or both.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_actor.c / rt_actor.h**: Actors query the `sprites[][]` spatial grid for collision/interaction with statics; call removal functions when destroying objects
- **rt_playr.c**: Player code picks up items (statics), triggers switches, and checks collision with hazardous statics
- **rt_game.c**: Main game loop calls `DoSprites()` and `AnimateWalls()` per frame
- **rt_draw.c / rt_view.c**: Rendering subsystem reads `statobj_t` fields (position, shapenum, flags) to render sprites and check visibility
- **rt_door.c**: Door/level transition code uses static objects for door visuals and transition triggers
- **rt_net.c**: Multiplayer sync serializes state via `SaveStatics()` / `LoadStatics()`
- **rt_debug.c**: Debug menu spawns test statics via `CheatSpawnItem()`
- **Cinematic subsystem** (cin_*.c): Scripted sequences may spawn/animate statics for cutscenes

### Outgoing (what this file depends on)
- **rt_ted.h**: Includes map structures, wall definitions, lighting state, spawn/door data, and tile constants (`MAPSIZE`, `MAPSPOT()`)
- **Lighting subsystem** (implicit): `ActivateLight()`, `TurnOnLight()` call into light rendering/management code
- **Texture/animation system** (implicit): `AnimateWalls()` and `statinfo.numanims` drive frame-based texture updates
- **Math/fixed-point library** (implicit): Uses `fixed` type for precise position arithmetic
- **Audio library** (implicit through respawn/activation callbacks): May trigger SFX on item pickup or hazard activation

## Design Patterns & Rationale

**Object Pool + Free List**: `statobj_t` instances are pre-allocated at level start. Active and inactive objects are maintained in separate doubly-linked lists (`firstactivestat`/`lastactivestat`, `firstemptystat`/`lastemptystat`), avoiding dynamic allocation during gameplay and enabling O(1) state transitions.

**Spatial Hashing**: The `sprites[MAPSIZE][MAPSIZE]` grid enables fast tile-based lookups (e.g., "what statics are at tile (10, 15)?"), critical for actor-static collision detection and player item pickup.

**Template/Lookup Tables**: Each of 91 object types has metadata in `stats[NUMSTATS]` (sprite ID, animation count, HP, damage, flags). This separates type definition from instance state, reducing memory footprint and enabling data-driven object creation.

**Separated Lifecycle**: Active/inactive lists allow the engine to update only in-use objects each frame, improving cache locality. Respawn queues (`respawn_t`) decouple object death from respawning, enabling configurable wait times.

**Rationale**: This design reflects late-1990s real-time constraints and DOS memory limits—object pooling and spatial hashing were essential for maintaining 60 FPS with limited RAM. The pattern is idiomatic to late-era software-rasterized engines.

## Data Flow Through This File

**Initialization Phase**:
- `InitStaticList()` → Populate free pool
- `InitAnimatedWallList()` → Prepare animation state
- Map loader calls `SpawnStatic(tilex, tiley, type, zoffset)` → allocate from pool, insert into `sprites[][]` and active list

**Per-Frame Update**:
- Game loop calls `DoSprites()` → iterate active list, decrement `ticcount`, handle animation/lifetime expiry, spawn respawn objects
- Game loop calls `AnimateWalls()` → update texture indices in `animwalls[]` for wall animations
- Renderer reads `sprites[][]` and active list → fetch `statobj_t.shapenum`, position, flags to draw

**Interaction**:
- Actor/player queries `sprites[tilex][tiley]` → find collidable statics at tile
- On item pickup: `RemoveStatic()` → unlink from lists, return to free pool
- On damage: `SpawnStaticDamage()` → spawn gibs as inert statics

**Persistence**:
- Save: `SaveStatics()`, `SaveAnimWalls()`, `SaveSwitches()` → serialize all state to buffer
- Load: `LoadStatics()` → reconstruct active/respawn lists and restore animation frame counters

## Learning Notes

**Idiomatic to This Engine**:
- **Fixed-point math** for deterministic physics and network sync (all positions use `fixed` type, not floats)
- **Tile-grid spatial partitioning** (map is divided into discrete tiles) rather than continuous octree/BVH
- **Linked lists everywhere** for dynamic data (actors, statics, effects) rather than modern array-of-structs or ECS
- **Type enums + lookup tables** (stat_t + statinfo[]) is a precursor to data-driven design; objects are mostly data-driven except for behavior

**Connections to Engine Concepts**:
- This is **not an ECS system**, but the statinfo[] pattern foreshadows it: type-based data lookup
- The active/inactive list pattern is a crude **entity management system** compared to modern engines, but achieves similar goals: culling, cache locality, and efficient iteration
- Spatial hashing (sprites[][]) is a form of **broad-phase collision** detection for statics; narrow-phase would be in actor or physics code
- Respawn queues are a **simple state machine** for object lifecycle (dead → respawning → alive)

**Modern Engines vs. This Code**:
- Modern engines use **ECS or unified object systems** (all entities, whether actor or static, are components); here they're separate hierarchies
- Modern engines use **continuous spatial structures** (BVH, quadtrees) rather than discrete tile grids
- Modern engines serialize via **reflection or serialization libraries**, not hand-coded buffer packing
- Modern engines animate via **timeline/keyframe systems** or **skeletal animation**, not frame counters on statinfo

## Potential Issues

- **No visible bounds checking** on array accesses (e.g., `sprites[tilex][tiley]` — assumes tilex/tiley are valid; no guards shown)
- **Linked-list traversal O(n)** for certain operations (e.g., finding an object by ID requires scanning a list rather than hash lookup)
- **Hardcoded limits** (`MAXSWITCHES=64`, `MAXANIMWALLS=17`, `NUMSTATS=91`) — expanding limits requires recompilation
- **Implicit dependencies** on `rt_ted.h` globals (e.g., map tile data, lighting state) make the subsystem tightly coupled to level representation
