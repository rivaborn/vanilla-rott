# rott/rt_door.c — Enhanced Analysis

## Architectural Role
This file serves as the **spatial state manager** for the game engine, controlling all interactive geometry (doors, push-walls, masked-walls) and the touch-plate/trigger system that links player actions to consequences. More critically, it maintains the `areaconnect[][]` connectivity matrix—a precomputed spatial graph that gates visibility, sound propagation, and AI perception across the level. By updating this matrix whenever doors or walls move, rt_door.c fundamentally controls which actors and statics are "active" and perceivable in each frame, making it essential for both gameplay fidelity and performance optimization.

## Key Cross-References

### Incoming (who depends on this file)
- **Main game loop** (`PlayLoop` context) calls `MoveDoors()`, `MovePWalls()`, `ProcessElevators()`, `TriggerStuff()`, `DoAnimatedMaskedWalls()` each frame
- **Actor system** (rt_actor.c) queries connectivity via `areabyplayer[]` to determine visibility and sound propagation; calls `ConnectAreas()` indirectly after door state changes
- **Stat system** (rt_stat.c) registers actions into touch-plates via `Link_To_Touchplate()` during level load; exports `ActivateLight`, `DeactivateLight` for action dispatch
- **Sound system** (rt_sound.h): rt_door.c calls `SD_PlaySoundRTP()`, `SD_PanRTP()`, `MU_StartSong()`, `MU_RestoreSongPosition()` when doors/elevators transition
- **Player input** (rt_playr.c) calls `OperateDoor()`, `OperatePushWall()` on key press
- **Network system** (rt_net.c) calls save/load functions for multiplayer persistence
- **Collision checks**: Actor movement (`ActorTryMove`, rt_actor.c) checks `actorat[][]` against door collision markers

### Outgoing (what this file depends on)
- **Memory management** (z_zone.h): `Z_LevelMalloc()`, `Z_Free()` for struct allocation
- **Map data structures** (extern globals): reads/writes `tilemap[][]`, `actorat[][]`, `mapplanes[][]`, `mapseen[][]` directly
- **Actor/stat linked lists** (rt_actor.c, rt_stat.c): iterates `firstactive`, `FIRSTSTAT`, `LASTSTAT`; accesses `objlist[]` for save-load pointer conversion; calls `MakeActive()`, `MakeInactive()`, `MakeStatActive()`, `MakeStatInactive()`
- **WAD system** (w_wad.h): `W_GetNumForName()`, `PreCacheLump()` to load textures
- **Sound pre-caching** (rt_sound.h): `SD_PreCacheSoundGroup()` during spawn
- **Game state** (rt_main.h): reads `gamestate`, `tics`, `loadedgame`, `insetupgame`
- **Player state** (rt_playr.h): reads `PLAYER[]` position for elevator and trigger checks
- **HUD/messaging** (rt_menu.h, rt_msg.h): calls `AddMessage()` for locked-door feedback
- **Random numbers** (rt_rand.h): `GameRandomNumber()` for push-wall marker logic
- **Debug helpers** (develop.h): `Error()`, `SoftError()`, `Debug()` with conditional output blocks

## Design Patterns & Rationale

**1. Global Fixed-Size Arrays with Index-Based Access**
- `doorobjlist[MAXDOORS]`, `pwallobjlist[MAXPWALLS]`, `maskobjlist[MAXMASKED]` store pointers; spawn functions increment `.._num` counters
- **Rationale**: Fast O(1) lookup by door/wall ID; fits procedural C86 era design; serializable by index alone
- **Tradeoff**: Fragile to array bounds; requires pre-allocation at compile-time

**2. Linked Lists for Active-Only Iteration**
- `FIRSTMASKEDWALL/LASTMASKEDWALL`, `FIRSTANIMMASKEDWALL/LASTANIMMASKEDWALL` hold only animating walls
- **Rationale**: Main loop only processes active (animating) masked walls, avoiding O(MAXMASKED) iteration
- **Modern equivalent**: ECS component queries or tagged entity collections

**3. Function Pointer Dispatch Table**
- `touchactions[NUMTOUCHPLATEACTIONS]` maps action indices to function pointers; `GetIndexForAction()` reverses lookup for save-load
- **Rationale**: Data-driven action binding—levels specify actions as indices, decoupled from code
- **Risk**: Rearranging array breaks existing save games; no type-safety on callbacks

**4. Area Connectivity Matrix + Reachability Bitmap**
- `areaconnect[NUMAREAS][NUMAREAS]` (precomputed door/wall-induced connectivity)
- `areabyplayer[NUMAREAS]` (cached reachability from player position via `RecursiveConnect()`)
- **Rationale**: Enables O(1) "is area visible to player?" queries; gates which actor/stat lists to process
- **Modern equivalent**: Binary Space Partition (BSP) or Portal Visibility System; modern engines compute real-time PVS

**5. Pointer-to-Index Serialization with Flags**
- Save-load converts actor/stat pointers to indices; uses `FL_TACT` (0x4000) and `FL_TSTAT` (0x8000) flags to disambiguate
- `LoadTouchPlates()` reverses: `if (dummy.whichobj & FL_TACT) temp->whichobj = (int)(objlist[...]);`
- **Rationale**: Pointers aren't stable across reload; indices are; flags allow single field to store different types
- **Risk**: If save format changes, pointers could be misinterpreted as actor vs. stat vs. raw int

**6. State Machines for Complex Transitions**
- Door: `dr_opening` → `dr_open` → `dr_closing` → `dr_closed`
- Elevator: `ev_ras` (ready at source) → `ev_mts` (moving to source) → `ev_doorclosing` → transitions
- Push-wall: `pw_npushed` → `pw_pushing` → `pw_moved` / `pw_moving`
- **Rationale**: Decouples animation frame updates from state logic; allows intermediate states (e.g., partial-open doors affect area connectivity)

## Data Flow Through This File

```
LEVEL LOAD:
  Map Parser calls SpawnDoor/SpawnPushWall/SpawnMaskedWall/Link_To_Touchplate
  → Objects stored in global arrays/linked lists
  → ConnectAreas() called once to populate areabyplayer[] based on initial state
  → Save/load restores all objects + calls FixDoorAreaNumbers/FixMaskedWallAreaNumbers

EACH FRAME (PlayLoop):
  MoveDoors() → updates animation, increments/decrements areaconnect[][]
  MovePWalls() → updates push-wall position, may call ConnectPushWall()
  ProcessElevators() → state machine; calls Teleport() if arrival
  TriggerStuff() → checks touchindices[player_tilex][player_tiley]
    → fires action callbacks (can modify door/light/object state)
    → actions trigger ConnectAreas()
  DoAnimatedMaskedWalls() → advance frame counters

PLAYER INTERACTION:
  Player presses use key on door:
    OperateDoor(keys, door_id, localplayer)
    → checks lock, gas barrier
    → UseDoor(door_id) → UtilizeDoor(door_id, OpenDoor/CloseDoor)
    → DoorOpening/DoorClosing update state over 20+ tics
    → first frame of DoorOpening calls ConnectAreas() (area now accessible)

KEY STATE TRANSITIONS:
  - Door open → areaconnect[area_a][area_b]++ (now connected)
  - Door closed → areaconnect[area_a][area_b]-- (disconnected)
  - ConnectAreas() walks areaconnect[][] from player area, marks reachable areas
  - Actors/statics with ~areabyplayer[] are deactivated (invisible, don't update)
  - Sounds only travel between connected areas
```

## Learning Notes

**Idiomatic to This Era (Early 90s):**
- Heavy reliance on global state; no encapsulation or OOP abstractions
- Procedural iteration (for loops over actor/stat lists) vs. modern ECS/event systems
- Fixed allocations & index-based data (no dynamic arrays; no memory pooling)
- Binary save format with pointer-to-index tricks; fragile if refactored
- Precomputed connectivity matrix instead of real-time spatial queries (PVS, ray-casting, etc.)

**Game Engine Concepts:**
- **Area/Portal System**: `areaconnect[][]` and `areabyplayer[]` form a simple portal visibility system; modern engines use PVS (Potentially Visible Set) or runtime portal culling
- **Touch Plates as Event System**: Early implementation of a trigger/action system; modern engines use message queues or event emitters (Unreal Blueprint, Unity Inspector callbacks)
- **Delayed Actions**: Touch plates support `tictime`/`ticcount` delays and queuing; shows awareness of frame-based deferred execution
- **Multi-Part Doors**: Linked doors (`DF_MULTI` flag) open/close in sync; shows level design flexibility for larger entrances
- **Push-Wall Secrets**: Auto-moving and marker-based push walls; sophisticated enough for hidden passages

**Surprising Sophistication:**
- Elevator system with locked doors, source/destination tracking, and automatic player teleportation + screen shake
- Animated masked-wall breaking with frame sequences and multi-wall groups
- Touch-plate action swapping (toggle behavior on repeated triggers)
- Area connectivity propagation for both actor visibility and audio

## Potential Issues

1. **Fixed Array Bounds**: `doorobjlist[MAXDOORS]`, `pwallobjlist[MAXPWALLS]` may overflow if level has too many; no guard in spawn functions
2. **Function Pointer Dispatch Fragility**: Rearranging `touchactions[]` breaks all saved games with touch-plate actions; no version/CRC check in save format
3. **Conditional Debug Output Affects Binary**: `#if LOADSAVETEST` sections include debug code in save/load path; if toggled, binary format changes (data corruption risk)
4. **Area Connectivity Recalculation Cost**: `ConnectAreas()` called on every door/wall state change; `RecursiveConnect()` could iterate all areas repeatedly if many connected doors toggle
5. **Pointer Cast Safety**: `(int)(objlist[...])` and `(int)(GetStatForIndex(...))` cast pointers to ints without checking alignment or null; UB on 64-bit systems
6. **Missing Bounds Validation**: `GetStatForIndex()` returns NULL if stat not found; callers may not check before dereferencing
7. **Global touchplate Indexing**: `lasttouch` byte can overflow past 255 touchplates; no assert on `Link_To_Touchplate()`
