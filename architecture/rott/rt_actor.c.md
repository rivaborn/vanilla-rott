# rott/rt_actor.c

## File Purpose
Implements the actor system for managing all dynamic game entities including enemies, projectiles, environmental hazards, and interactive objects. Handles spawning, physics, collision, damage, state management, AI behavior, and save/load persistence for the game world.

## Core Responsibilities
- **Actor lifecycle management**: Allocation, initialization, cleanup, and free list maintenance for actor objects
- **Physics simulation**: Gravity, momentum, friction, collision detection and response, Z-axis movement
- **State machine execution**: State transitions, think function dispatch, animation frame advancement per actor class
- **Collision system**: Multi-stage collision checks (actors, walls, doors, static objects, masked walls) with response generation
- **Damage and death**: Player/enemy damage handling, fatality sequences, gib spawning, death state transitions
- **Projectile system**: Missile spawning, movement, collision detection, area-of-effect explosions, hitscan weapon traces
- **AI behavior**: Pathfinding, chase logic, attack decision-making, special boss behaviors (Darian, Heinrich, Oscuro)
- **Spatial organization**: Per-area actor linked lists, tile-based actor map, active/inactive actor tracking
- **Save/restore state**: Actor serialization/deserialization for game persistence

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `objtype` | struct | Main actor/object entity with position, velocity, state, health, flags |
| `playertype` | struct | Player-specific state (health, inventory, animation frame) linked from objtype |
| `statobj_t` | struct | Static sprite objects (decorations, pickups, explosions) |
| `saved_actor_type` | struct | Serialized actor data format for save files |
| `basic_actor_sounds` | struct | Sound IDs for actor classes (see/fire/pain/die) |
| `doorobj_t` | struct | Door entity (position, state, lock status) |
| `wall_t` | struct | Wall tile with physics and damage properties |
| `maskedwallobj_t` | struct | Multi-layer textured wall (glass, platforms, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FIRSTACTOR`, `LASTACTOR` | objtype* | global | Head/tail pointers for actor doubly-linked list |
| `FIRSTFREE`, `LASTFREE` | objtype* | global | Free actor pool for recycling |
| `firstactive`, `lastactive` | objtype* | global | Active actor list for per-frame think calls |
| `firstareaactor[NUMAREAS+1]` | objtype* | global | Per-area actor head pointers (spatial partitioning) |
| `lastareaactor[NUMAREAS+1]` | objtype* | global | Per-area actor tail pointers |
| `actorat[MAPSIZE][MAPSIZE]` | void* | global | Spatial map: grid cell → actor/wall/sprite for collision queries |
| `objcount` | int | global | Current number of active actors allocated |
| `new` | objtype* | global | Currently allocated actor (temporary variable) |
| `MISCVARS`, `mstruct` | misc_stuff* | global | Miscellaneous game state (gibs, noise, modes) |
| `BAS[NUMCLASSES+3]` | basic_actor_sounds[] | static | Sound lookup table indexed by actor class |
| `starthitpoints[4][NUMENEMIES+2]` | int[][] | static | Enemy base health by difficulty×class |
| `UPDATE_STATES[NUMSTATES][NUMENEMIES]` | statetype*[][] | static | State table for state machine transitions |
| `angletodir[ANGLES]` | int[] | global | Angle → 8-dir mapping for AI pathfinding |
| `SNAKEHEAD`, `SNAKEEND` | objtype* | static | Head/tail of dark snake boss segment chain |
| `PARTICLE_GENERATOR`, `EXPLOSIONS` | objtype* | static | Special tracked actors for particle/explosion systems |
| `ludicrousgibs` | boolean | global | Enables excessive gore mode |
| `Masterdisk` | boolean | static | Elevator disk master flag |
| `STOPSPEED`, `PLAYERFRICTION`, `ACTORFRICTION` | int | static | Physics constants for momentum and deceleration |
| `MissileSound` | boolean | static | Controls whether projectile sounds play (muting for rapid fire) |

## Key Functions / Methods

### DoActor
- **Signature**: `void DoActor(objtype *ob)`
- **Purpose**: Main per-frame update for a single actor; core game loop hook
- **Inputs**: `ob` — actor to process
- **Outputs/Return**: None; modifies actor state and may remove actor
- **Side effects**: Calls actor's think function, advances animation frame, updates position, marks spatial cell
- **Calls**: ApplyGravity, M_CheckDoor, M_CheckBossSounds, ControlPlayerObj, NewState, RemoveObj
- **Notes**: Called once per actor per frame; if actor state becomes NULL, actor is removed

### ActorMovement
- **Signature**: `void ActorMovement(objtype *ob)`
- **Purpose**: Apply momentum, friction, and collision to move actor; dispatches to ActorTryMove
- **Inputs**: `ob` — actor to move
- **Outputs/Return**: None; updates actor position
- **Side effects**: Modifies momentum, position; may call MoveActor, PlayerSlideMove, calls movement checks
- **Calls**: ActorTryMove, MoveActor, ApplyGravity, FindDistance, FixedMul, FixedDiv2
- **Notes**: Enforces max speed, applies friction based on actor flags/state; handles riding on platforms

### ActorTryMove
- **Signature**: `boolean ActorTryMove(objtype*ob, int tryx, int tryy, int tryz)`
- **Purpose**: Test if actor can move to new position; main collision dispatcher
- **Inputs**: `ob` — actor; `tryx, tryy, tryz` — proposed new position in fixed-point
- **Outputs/Return**: true if move succeeds, false if blocked
- **Side effects**: May modify actor z-position during stepping; adjusts momentum on collision
- **Calls**: CheckOtherActors, CheckRegularWalls, CheckStaticObjects, CheckMaskedWalls, CheckDoors
- **Notes**: Multi-phase collision check; returns immediately on first blocking layer

### Collision
- **Signature**: `void Collision(objtype*ob, objtype *attacker, int hitmomentumx, int hitmomentumy)`
- **Purpose**: Handle collision response between two actors (damage, knockback, state change)
- **Inputs**: `ob` — victim; `attacker` — cause of collision; hit momentum values
- **Outputs/Return**: None; updates victim flags and may transition state
- **Side effects**: Damages actor, may trigger fatality sequence, applies momentum
- **Calls**: ActivateEnemy, BeginEnemyFatality, BeginEnemyHurt, BeginPlayerFatality, M_CheckPlayerKilled
- **Notes**: Checks FL_SHOOTABLE flag before applying damage; different responses for player vs enemy

### DamageThing
- **Signature**: `void DamageThing(void *thing, int damage)`
- **Purpose**: Universal damage dispatcher; routes to appropriate damage handler
- **Inputs**: `thing` — actor or sprite pointer; `damage` — hit points to subtract
- **Outputs/Return**: None; updates target hitpoints
- **Side effects**: May trigger death sequences, remove statics, spawn gibs
- **Calls**: DamagePlayerActor, DamageNonPlayerActor, DamageStaticObject
- **Notes**: Checks if target is FL_SHOOTABLE before allowing damage; uses which field to distinguish object type

### SpawnMissile
- **Signature**: `void SpawnMissile(objtype* shooter, classtype nobclass, int nspeed, int nangle, statetype*nstate, int offset)`
- **Purpose**: Allocate and initialize a projectile/missile
- **Inputs**: shooter, missile class, speed, angle, initial state, spawn offset from shooter
- **Outputs/Return**: None; sets global `new` to created missile
- **Side effects**: Allocates actor; plays sound; sets momentum and special temp variables
- **Calls**: GetNewActor, MakeActive, Set_3D_Momenta, ParseMomentum, NewState, SD_PlaySoundRTP
- **Notes**: Handles both hitscan and projectile weapons; automatically plays fire sound unless MissileSound suppressed

### MissileHit
- **Signature**: `void MissileHit(objtype *ob, void *hitwhat)`
- **Purpose**: Handle projectile collision with world/actor/static
- **Inputs**: `ob` — missile; `hitwhat` — collision target (actor/wall/sprite/NULL for environment)
- **Outputs/Return**: None; transitions missile to explosion state
- **Side effects**: Spawns explosion, applies damage, triggers special effects (knockback, fire spread)
- **Calls**: MissileHitActor, NewState, SpawnFirebomb, SD_PlaySoundRTP
- **Notes**: Damage varies by missile type and collision target; some missiles have special behaviors (split, spread)

### SaveActors / LoadActors
- **Signature**: `void SaveActors(byte **buffer, int*size)` / `void LoadActors(byte *buffer, int size)`
- **Purpose**: Serialize/deserialize all actors for save file
- **Inputs**: Buffer pointer/size for serialization
- **Outputs/Return**: Fills buffer with actor data; updates size; restores actor lists from buffer
- **Side effects**: Allocates memory; rebuilds actor links and spatial maps
- **Calls**: GetNewActor, InitActorList, GetIndexForState, GetStateForIndex, GetStaticForIndex
- **Notes**: Handles actor-to-actor and actor-to-static references via index tables; manages two-pass linking

### InitActorList
- **Signature**: `void InitActorList(void)`
- **Purpose**: Initialize all actor lists and globals at level start
- **Inputs**: None
- **Outputs/Return**: None; zeros all actor tracking structures
- **Side effects**: Clears actor pools, resets counters, calls FindAddresses
- **Calls**: memset, FindAddresses
- **Notes**: Must be called before any actor operations; resets MISCVARS and particle/explosion globals

### NewState
- **Signature**: `void NewState(objtype *ob, statetype *newstate)`
- **Purpose**: Transition actor to new behavior state
- **Inputs**: `ob` — actor; `newstate` — target state pointer
- **Outputs/Return**: None; updates state and animation frame
- **Side effects**: Updates ticcount from state, sets shapenum for rendering
- **Calls**: SetVisiblePosition, DoPanicMapping (conditional)
- **Notes**: Panic mapping may redirect explosions to alternate states; ticcount is (tictime >> 1) to account for rendering double-buffering

### RayShoot
- **Signature**: `void RayShoot(objtype * shooter, int damage, int accuracy)`
- **Purpose**: Cast a line-of-sight ray from hitscan weapon; find and damage first hit target
- **Inputs**: shooter actor, damage amount, accuracy (affects spread)
- **Outputs/Return**: None; spawns gun smoke, calls appropriate hit function
- **Side effects**: Damages actors/sprites in ray path, updates illumination, spawns visual effects
- **Calls**: ShootActor, ShootSprite, SpawnGunSmoke, SetIllumination, CheckLine
- **Notes**: 3D ray with Z-angle support; stops at first blocking surface (wall/actor); handles bullet holes per wall section

---

## Control Flow Notes
**Initialization**: `InitActorList()` called at level load; static state table `UPDATE_STATES` provides state transitions by actor class.

**Per-Frame Update**: Game loop calls `DoActor()` for each actor in linked list:
1. Apply physics (gravity, momentum)
2. Dispatch think function from current state
3. Update animation frame
4. Mark in spatial map for rendering

**Collision Resolution**: Triggered by `ActorMovement()` → `ActorTryMove()` → multi-phase checks → `Collision()` for response.

**Damage/Death Flow**: Weapons fire → `RayShoot()` or `MissileHit()` → `DamageThing()` → `BeginEnemyFatality()` / `BeginPlayerFatality()` → state transition to death sequence.

**Cleanup**: `RemoveObj()` returns actor to free pool; `SaveActors()` / `LoadActors()` handle persistence across save/load.

## External Dependencies
- **rt_def.h**: Core type definitions (`objtype`, `classtype`, `exit_t`, flag constants)
- **rt_sound.h**: Sound playback (`SD_PlaySoundRTP`, `SD_StopSound`, `SD_SoundActive`)
- **rt_door.h**: Door/elevator structures, collision types (`doorobj_t`, `maskedwallobj_t`)
- **rt_ted.h**: Level data access (`MAPSPOT`, `AREANUMBER`, platform checks)
- **rt_draw.h**: Rendering support (`SetVisiblePosition`, `SetFinePosition`, lighting)
- **states.h**: State table definitions (`statetype`, state pointers like `&s_chase1`)
- **sprites.h**: Sprite/animation management (`TurnActorIntoSprite`, `PreCacheActor`)
- **gmove.h**: Movement utilities (`FindDistance`, `atan2_appx`, angle/direction tables)
- **rt_game.h**: Game state (`gamestate`, `MISCVARS`, `PLAYER[0]`)
- **Defined elsewhere**: `FIRSTSTAT`, `LASTSTAT` (sprite list); `doorobjlist[]`, `maskobjlist[]`, `pwallobjlist[]` (object pools); `tilemap[][]` (collision grid); `costable/sintable` (trig lookup)
