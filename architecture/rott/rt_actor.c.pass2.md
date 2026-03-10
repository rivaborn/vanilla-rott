# rott/rt_actor.c — Enhanced Analysis

## Architectural Role

`rt_actor.c` is the **central entity management hub** for the ROTT engine—it's responsible for managing every dynamic game object's lifecycle, physics, AI, and interaction. Every actor (enemy, projectile, hazard, effect) flows through this file's state machine, collision system, and serialization layers. Because actors are spatially partitioned by area and rendered through a unified pipeline, this file serves as the bridge between the game logic layer (states, AI, damage) and the rendering/collision substrate (rt_draw, rt_door, rt_ted). The actor system is intrinsically linked to the **sound subsystem** (damage/death sounds), **sprite system** (rendering, animation), and **player controller** (collision with player, damage, knockback).

## Key Cross-References

### Incoming (who depends on this file)

From includes and globals, these subsystems call into rt_actor.c:

- **rt_game.c**: Game loop calls `DoActor()` per-frame for each active actor; also calls `InitActorList()` at level load
- **rt_playr.c**: Player update logic reads/writes actor globals (`FIRSTACTOR`, `LASTACTOR`, `PLAYER0MISSILE`) for collision and weapon feedback
- **rt_door.c**: Door/elevator logic reads actor spatial lists to detect actors blocking doors, reads `actorat[][]` grid
- **rt_ted.c**: Level loading spawns actors via `GetNewActor()`; queries area numbers via `firstareaactor[]` lists
- **rt_stat.c**: Static sprite system reads `actorat[][]` to check for actor occupancy; calls `DamageThing()` for destructible statics
- **rt_draw.c**: Rendering reads visible actor positions set by `SetVisiblePosition()`, reads state animation frames
- **rt_menu.c**: Save/load calls `SaveActors()` and `LoadActors()` for persistence
- **rt_net.c**: Network multiplayer calls `ControlPlayerObj()` for remote player updates, serializes actor state
- **cin_actr.c**: Cinematic system spawns actors and manages their state for cutscenes
- **rt_util.c**: Utility functions read angletodir table, query actor spatial state
- **rt_debug.c** (dev): Debug mode reads actor globals for inspection

### Outgoing (what this file depends on)

- **rt_sound.h**: Calls `SD_PlaySoundRTP()` for positional audio, `SD_StopSound()` for silence effects
- **rt_draw.h**: Calls `SetVisiblePosition()`, `SetFinePosition()` for rendering, `CalcHeight()` for elevation
- **rt_door.h**: Calls `M_CheckDoor()` for elevator/door collision; reads `doorobjlist[]`, `maskobjlist[]`, `pwallobjlist[]` arrays
- **rt_ted.h**: Queries `MAPSPOT()`, `AREANUMBER()`, `tilemap[][]` for level geometry
- **states.h**: Reads state table pointers (`&s_chase1`, etc.); dispatches `state->think()` function pointers
- **sprites.h**: Calls `TurnActorIntoSprite()` for death-to-sprite conversions, `PreCacheActor()` for animation pre-loading
- **gmove.h**: Uses `FindDistance()`, `atan2_appx()`, `costable/sintable` for navigation, angle calculations
- **rt_game.h**: Reads `gamestate`, `MISCVARS`, `PLAYER[0]` for game context; reads difficulty for health tables
- **rt_floor.h** (implied): Platform/floor queries for actor elevation handling
- **rt_view.c**: May call for camera/eye object positioning
- **rt_net.c**: Network packet serialization; reads/writes actor state over network
- **rt_cfg.c**: Reads configuration for gameplay constants (friction, speed caps)

## Design Patterns & Rationale

1. **Linked-List Object Pool**: Actor free/active lists + per-area sublists (firstareaactor) reduce allocation/deallocation churn; recycling objects is cache-friendly and deterministic. This is common in 1990s game engines before generational GC.

2. **Spatial Hashing via Tile Map (`actorat[][]`)**: Every tile cell stores a pointer to actor/wall/sprite occupying it. This enables O(1) collision queries instead of O(n) sweeps. Per-area lists add hierarchical spatial organization for visibility culling.

3. **State Machine via Lookup Table (`UPDATE_STATES[NUMSTATES][NUMENEMIES]`)**: Decouples state logic from class logic; state transitions are data-driven. This pattern scales to many enemy types without branching on class. Shareware/full builds use conditional compilation to swap tables.

4. **Two-Pass Save/Load**: Actor references (missile→shooter, etc.) are saved as indices and restored by a second-pass link phase. Avoids pointer serialization; robust to memory layout changes.

5. **Actor-to-Actor Communications via Temporaries**: The global `new` variable holds the newly-allocated actor after `SpawnMissile()`, allowing callers to configure it. Similarly, `MISCVARS` holds temporary state (gib counts, noise values) across function calls without deep parameter passing.

6. **Difficulty-Based Stat Scaling**: `starthitpoints[difficulty][class]` table allows balancing without code changes; hardcoded in data.

**Why this design?** ROTT targets mid-1990s hardware: no dynamic memory fragmentation, cache-efficient list traversal, minimal per-frame allocation. Lookup tables (states, sounds, hitpoints) compress code size and make tweaking art/balance easy without recompilation.

## Data Flow Through This File

1. **Spawn**: 
   - Level loader (rt_ted) → `GetNewActor()` → allocate from free pool, insert into FIRSTACTOR list and per-area sublist
   - `NewState()` initializes animation frame from state table
   
2. **Per-Frame Update** (`DoActor()`):
   - Actor's state.think() function is called (enemy AI, projectile movement, hazard behavior)
   - `ActorMovement()` applies momentum + friction → `ActorTryMove()` (multi-phase collision) → response (knockback, damage)
   - Animation frame advances; actor marked in `actorat[][]` for next collision check
   
3. **Collision Resolution** (`ActorTryMove()`):
   - Phase 1: `CheckOtherActors()` — actors collide with actors
   - Phase 2: `CheckRegularWalls()` — wall geometry stops movement
   - Phase 3: `CheckStaticObjects()` — sprite obstacles
   - Phase 4: `CheckMaskedWalls()` — textured/translucent barriers
   - Phase 5: `CheckDoors()` — elevators and sliding doors
   - First hit stops movement; momentum adjusted
   
4. **Damage/Death**:
   - Weapon fire → `RayShoot()` or missile → `MissileHit()` → `DamageThing()`
   - `Collision()` checks if target is shootable, applies damage
   - If hitpoints ≤ 0 → `BeginEnemyFatality()` / `BeginPlayerFatality()` → transition to death state
   - Death state may spawn gibs (sprites) or trigger special death animation
   
5. **Cleanup**:
   - State becomes NULL or actor time expires → `RemoveObj()` → return to free pool
   - `SaveActors()` before level end; `LoadActors()` on restore

## Learning Notes

**For engine developers studying this file:**

- **Idiomatic to this era**: Lookup tables for state machines, pre-allocated object pools, tile-based spatial hashing, and global temporaries (`new`, `MISCVARS`) were standard because memory was precious and dynamic allocation unreliable. Modern engines use ECS (Entity-Component-System), generational arenas, or scene graphs instead.
  
- **Actor vs. Sprite distinction**: rt_actor.c manages dynamic entities with physics and AI; rt_stat.c manages static sprites (decorations, gibs). This separation makes sense: actors have complex state machines; sprites are fire-and-forget visuals. Some modern engines merge them as Entities.

- **State Machine Insight**: The `UPDATE_STATES` table is a clever way to express behavior trees without explicit code. Each actor class has a state for each behavior (stand, path, chase, shoot, die). The state function pointers (in states.h) then implement the actual logic. This is close to a classic **Behaviour Tree** pattern but simpler.

- **Spatial Partitioning**: `actorat[][]` + `firstareaactor[]` is a hybrid approach: coarse grid per tile + fine linked lists per area. This allows fast collision checks without full quadtree complexity. Modern engines use broad-phase (AABB trees) + narrow-phase (SAT/GJK).

- **No Component-Based Design**: Damage, hitpoints, sounds are all baked into `objtype` struct. A modern ECS would split these into separate components (Health, Audio, Physics) and query systems. This monolithic approach is simpler but less flexible.

- **Deterministic Ordering**: Actors update in linked-list order, no sorting by priority. This can cause frame-order bugs (actor A damages B, then B acts this frame—unfair if A goes first). Some engines use fixed update order or depth sorting.

## Potential Issues

1. **Missile Duplication Risk**: `SaveActors()` saves missiles by index; if the same missile is referenced by multiple actors (shooter + owner), the restore may create duplicates or broken pointers. The two-pass linking helps but isn't foolproof.

2. **Collision Grid Staleness**: `actorat[][]` is populated per actor in `DoActor()`, but if actors are removed or repositioned mid-frame without clearing their old cells, stale pointers persist. The code assumes actors update in order and grid is refreshed each frame.

3. **Circular References in Chains**: `SNAKEHEAD` / `SNAKEEND` form a linked chain for the boss segmented body. If a segment dies mid-chain, iterating from head may hit a NULL or dangling pointer. Special case handling exists but is fragile.

4. **Sound Cutoff via MissileSound Flag**: `MissileSound` is a global toggle to prevent rapid-fire weapons from saturating audio. If multiple callers toggle this, race conditions can mute unintended sounds. A sound priority queue would be more robust.

5. **Difficulty-Scaling Inconsistency**: `starthitpoints` table has 4 difficulty levels but some bosses have hardcoded `-1` (undefined). If a boss is spawned on that difficulty, `SHP()` macro will dereference `-1`, causing crashes. The code assumes all difficulty×class combinations are populated.

6. **Area Number Validation**: `ValidAreanumber()` checks `0 <= n <= NUMAREAS`, but actor lists are `firstareaactor[NUMAREAS+1]` (index 0 to NUMAREAS inclusive). If an actor's area is set to NUMAREAS+1 or higher, it's silently ignored, potentially losing the actor.

---

**Overall**, rt_actor.c is a masterclass in pragmatic 1990s game engine design: simple, fast, and highly coupled to the rendering and collision pipelines. Its state machine abstraction is elegant; its spatial partitioning is effective. The main "issues" are really design trade-offs (monolithic types, global temporaries) that modern engines have solved with more abstraction, but for its era and hardware constraints, this design is sound.
