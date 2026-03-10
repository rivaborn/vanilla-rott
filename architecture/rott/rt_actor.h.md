# rott/rt_actor.h

## File Purpose
Public header for the actor (game entity) system in ROTT. Defines the core `objtype` structure representing any dynamic object (enemies, player, projectiles, hazards), declares spawn/despawn/movement/collision functions, and provides macros for actor state management and position queries.

## Core Responsibilities
- Define actor object structure (`objtype`) and classify entities by type (`classtype` enum)
- Manage actor linked lists (FIRSTACTOR/LASTACTOR, area actors, active/inactive actors)
- Spawn and despawn actors (enemies, hazards, projectiles, particles)
- Provide position manipulation macros (tile-based and fine-grained)
- Handle collision detection and response (ActorTryMove, MissileTryMove, Collision)
- Damage calculation and actor death (DamageThing, KillActor, MissileHit)
- State machine integration (NewState, DoActor) for AI and behavior
- Pathfinding and chase logic (SelectPathDir, SelectChaseDir, SightPlayer)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `objtype` | struct | Central actor/object structure; contains position, state, physics, linked-list pointers, flags |
| `classtype` | enum | Classifies all actor types: enemies (lowguard, highguard, etc.), bosses (Darian, Heinrich, etc.), hazards (spear, blade, boulder), projectiles (shuriken, grenade, missile), collectibles |
| `statetype` | struct | State machine node (defined in states.h); holds animation frame, tic duration, think function, and next state pointer |
| `basic_actor_sounds` | struct | Sound handles for actor behaviors: operate, see, fire, hit, die |
| `misc_stuff` | struct | Miscellaneous game state: gib settings, touch/collision tracking, enemy counts, door state, actor deletion queue |
| `_2Dpoint` | struct | Simple 2D integer point (x, y) |
| `gib_t` | enum | Gore/gib particle types: sparks, organs, limbs, souls, spit, etc. |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FIRSTACTOR`, `LASTACTOR` | objtype* | global | Head/tail of linked list of all actors in play |
| `PLAYER0MISSILE` | objtype* | global | Cached reference to player's active missile (for updates) |
| `FIRSTRAIN`, `LASTRAIN` | objtype* | global | Linked list of rain particle actors |
| `SCREENEYE` | objtype* | global | Cached reference to screen camera/eye actor |
| `firstareaactor[]`, `lastareaactor[]` | objtype* | global | Per-area actor lists (NUMAREAS+1 buckets) for spatial partitioning |
| `mstruct`, `MISCVARS` | misc_stuff | global | Aggregate miscellaneous game state (gib settings, touch queue, counts) |
| `actorat[MAPSIZE][MAPSIZE]` | void* | global | Spatial grid: each tile maps to actor occupying it (collision lookup) |
| `firstactive`, `lastactive` | objtype* | global | Linked list of active (moving/thinking) actors vs. static ones |
| `new`, `objlist`, `killerobj` | objtype* | global | Actor pool and last-hit tracking for damage resolution |
| `angletodir[]` | int | global | Lookup table converting angles to 8-directional direction indices |
| `SNAKEPATH[]` | _2Dpoint | global | Pre-computed path waypoints for snake-like boss movement |
| `ludicrousgibs` | boolean | global | Flag enabling extreme gore effects |
| `objcount` | int | global | Counter of actors in play |

## Key Functions / Methods

### SpawnInertActor
- **Signature:** `void SpawnInertActor(int tilex, int tiley, int type)`
- **Purpose:** Create a static/decorative actor at a tile position.
- **Inputs:** tilex, tiley (tile grid coords); type (class identifier)
- **Outputs/Return:** None; modifies global actor list
- **Side effects:** Allocates actor, inserts into lists, updates spatial grid
- **Calls:** GetNewActor (implied), list insertion macros
- **Notes:** "Inert" suggests non-moving, non-thinking

### ActorTryMove
- **Signature:** `boolean ActorTryMove(objtype* ob, int tilex, int tiley, int dir)`
- **Purpose:** Attempt to move actor to new tile; handle collision/blocking.
- **Inputs:** ob (actor pointer), tilex/tiley (target tile), dir (direction for edge logic)
- **Outputs/Return:** true if move succeeds; false if blocked
- **Side effects:** Updates ob->tilex, ob->tiley, ob->x, ob->y if successful; may trigger collisions
- **Calls:** QuickSpaceCheck, collision logic (not visible in this file)
- **Notes:** Core pathfinding step; respects walls, actors, and hazards

### MissileTryMove
- **Signature:** `boolean MissileTryMove(objtype* ob, int tilex, int tiley, int z)`
- **Purpose:** Move projectile/missile, handling 3D (z-axis) and collision.
- **Inputs:** ob (projectile actor), tilex/tiley (target tile), z (altitude)
- **Outputs/Return:** true if move succeeds; false if hit obstacle
- **Side effects:** Updates position; may trigger missile impact (MissileHit)
- **Calls:** Collision detection, MissileHit (on impact)
- **Notes:** Distinct from ActorTryMove due to 3D physics and special impact logic

### DoActor
- **Signature:** `void DoActor(objtype* ob)`
- **Purpose:** Main per-frame update loop for a single actor; advance state machine and execute think functions.
- **Inputs:** ob (actor to update)
- **Outputs/Return:** None
- **Side effects:** Increments ticcount; calls state->think() function; may transition states via NewState
- **Calls:** NewState, state machine think functions (T_Chase, T_Path, A_Shoot, etc.)
- **Notes:** Called once per actor per frame; gateway to AI behavior

### NewState
- **Signature:** `void NewState(objtype* ob, statetype* state)`
- **Purpose:** Transition actor to a new state machine node; reset animation/timing.
- **Inputs:** ob (actor), state (new state to enter)
- **Outputs/Return:** None
- **Side effects:** Updates ob->state, resets ob->ticcount to 0; may call state->think() immediately
- **Calls:** state->think() (if defined)
- **Notes:** State pointer is immutable after set until next NewState; ticcount drives animation

### T_Chase, T_Path, T_Stand, T_Projectile
- **Signature:** `void T_Chase(objtype* ob)`, `void T_Path(objtype* ob)`, `void T_Stand(objtype* ob)`, `void T_Projectile(objtype* ob)`
- **Purpose:** Think functions for specific behaviors: chase player, follow patrol path, idle standing, fly/roll.
- **Inputs:** ob (actor executing behavior)
- **Outputs/Return:** None
- **Side effects:** Call ActorTryMove, SelectChaseDir, etc.; may call NewState to transition out
- **Calls:** ActorTryMove, SelectChaseDir, SelectPathDir, KillActor, collision checks
- **Notes:** Called from DoActor's state machine; control AI decision-making

### SelectChaseDir, SelectPathDir
- **Signature:** `void SelectChaseDir(objtype* ob)`, `void SelectPathDir(objtype* ob)`
- **Purpose:** Calculate and set movement direction toward player (chase) or next waypoint (path).
- **Inputs:** ob (actor)
- **Outputs/Return:** None
- **Side effects:** Updates ob->dir (direction enum), ob->angle, ob->yzangle
- **Calls:** AngleBetween, SightPlayer, trigonometric/lookup tables
- **Notes:** Direction determines next ActorTryMove attempt

### SightPlayer, CheckSight, CheckLine
- **Signature:** `boolean SightPlayer(objtype* ob)`, `boolean CheckSight(objtype* ob, void* target)`, `boolean CheckLine(void* obj1, void* obj2, int type)`
- **Purpose:** Line-of-sight checks for AI perception and projectile hit detection.
- **Inputs:** ob/target (actor pointers); type (line-trace mode)
- **Outputs/Return:** true if line unobstructed; false if blocked by wall/actor
- **Side effects:** None (read-only queries)
- **Calls:** Map geometry queries (not visible in this file)
- **Notes:** Essential for AI activation and target validation

### SpawnMissile, MissileHit
- **Signature:** `void SpawnMissile(objtype* shooter, classtype type, int tilex, int tiley, statetype* state, int angle)`, `void MissileHit(objtype* missile, void* target)`
- **Purpose:** Create and fire projectile (shuriken, grenade, etc.); handle hit resolution.
- **Inputs:** shooter (source actor), type (missile class), tilex/tiley (spawn tile), state (initial state), angle (fire direction); target (hit object)
- **Outputs/Return:** None
- **Side effects:** Allocates missile actor; triggers hit effects, damage, particle spawns
- **Calls:** SpawnNewObj, DamageThing, SpawnParticles, ParticleEffects
- **Notes:** Missiles are actors with T_Projectile think; hit callback resolves impact

### DamageThing, KillActor
- **Signature:** `void DamageThing(void* victim, int damage)`, `void KillActor(objtype* ob)`
- **Purpose:** Apply damage to any hittable object (actor or player); transition to death/destruction.
- **Inputs:** victim (void pointer to actor/thing), damage (int hitpoints); ob (actor to destroy)
- **Outputs/Return:** None
- **Side effects:** Decrements hitpoints; may call NewState to death state; may spawn gibs/particles; updates score
- **Calls:** NewState (death transition), SpawnParticles, M_CheckPlayerKilled
- **Notes:** DamageThing is flexible (void*) for polymorphism; KillActor is immediate destruction

### A_Shoot, A_MissileWeapon
- **Signature:** `void A_Shoot(objtype* ob)`, `void A_MissileWeapon(objtype* ob)`
- **Purpose:** Action functions triggered by state machine to fire hitscan (bullet) or projectile weapons.
- **Inputs:** ob (shooting actor)
- **Outputs/Return:** None
- **Side effects:** Spawns projectile actor; calls RayShoot or SpawnMissile; plays fire sound
- **Calls:** SpawnMissile, RayShoot, sound system, NewState (to recoil/recovery state)
- **Notes:** Called by think functions; respects ammunition and cooldowns via state transitions

### Collision, T_Collide
- **Signature:** `void Collision(objtype* ob, objtype* attacker, int hitmomentumx, int hitmomentumy)`, `void T_Collide(objtype* ob)`
- **Purpose:** Handle physical collision with another actor (knockback, damage); resolve blocked movement.
- **Inputs:** ob (affected actor), attacker (source of collision), momentum (velocity imparted)
- **Outputs/Return:** None
- **Side effects:** Updates ob->momentumx/y; may call DamageThing or NewState; updates velocities
- **Calls:** DamageThing, NewState, physics integration
- **Notes:** T_Collide is a think function for collision response states; Collision is called during movement resolution

### MakeActive, MakeInactive
- **Signature:** `void MakeActive(objtype* ob)`, `void MakeInactive(objtype* ob)`
- **Purpose:** Transition actor between active (thinking/moving) and inactive (static) lists.
- **Inputs:** ob (actor)
- **Outputs/Return:** None
- **Side effects:** Moves actor pointer between firstactive/lastactive and static pool
- **Calls:** List manipulation (not visible)
- **Notes:** Optimization: inactive actors skip DoActor calls

### RemoveObj
- **Signature:** `void RemoveObj(objtype* ob)`
- **Purpose:** Deallocate actor and remove from all lists.
- **Inputs:** ob (actor to remove)
- **Outputs/Return:** None
- **Side effects:** Unlinks from FIRSTACTOR/LASTACTOR, area lists, active lists; updates spatial grid; queues for deferred deletion
- **Calls:** RemoveFromArea, MakeInactive, Add_To_Delete_Array
- **Notes:** May defer deletion (Add_To_Delete_Array) to avoid iterator invalidation

### InitActorList
- **Signature:** `void InitActorList(void)`
- **Purpose:** Clear and reinitialize actor system at level start.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets FIRSTACTOR/LASTACTOR, clears actorat grid, resets objcount
- **Calls:** Memory init (not visible)
- **Notes:** Called during level load before spawning entities

---

## Control Flow Notes

**Initialization:** InitActorList() clears state at level load.

**Spawn:** Game code calls SpawnInertActor, SpawnStand, SpawnPatrol, SpawnMissile, etc., to create entities. These allocate from a pool, insert into FIRSTACTOR/LASTACTOR and area buckets, and set initial state.

**Per-Frame Update:** 
1. Iterate FIRSTACTOR → LASTACTOR (or firstactive → lastactive for active only).
2. Call DoActor(ob) for each actor.
3. DoActor increments ticcount; if ticcount ≥ state→tictime, call state→think() and reset ticcount.
4. Think functions (T_Chase, T_Path, A_Shoot, etc.) call ActorTryMove, SelectChaseDir, or NewState to advance behavior.
5. Movement calls may trigger Collision or MissileHit, which applies DamageThing or KillActor.

**Cleanup:** RemoveObj unlinks actor and queues for deferred deletion; Remove_Delete_Array_Entries is called at end-of-frame to actually deallocate (avoids iterator corruption).

**Render:** drawx, drawy fields (set by SetVisiblePosition macro) are consumed by renderer; not updated by this file.

## External Dependencies
- **Include:** `states.h` — State machine structures (statetype) and state instance externs (s_lowgrdstand, s_chase1, etc.)
- **Defined elsewhere (used here):**
  - `statetype` struct and state instances
  - `thingtype`, `dirtype` enums (basic types)
  - BATTLE_CheckGameStatus, sound system functions
  - TILESHIFT, TILEGLOBAL, MAPSIZE constants (map module)
  - Memory allocator (GetNewActor)
  - Trigonometric/angle utilities (angletodir[], AngleBetween)
