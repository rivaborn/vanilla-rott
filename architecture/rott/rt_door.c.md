# rott/rt_door.c

## File Purpose
Manages all door, elevator, push-wall, masked-wall, and touch-plate (trigger) systems for the game. Handles spatial state (open/closed, area connectivity), player interaction, and save/load persistence of these interactive elements.

## Core Responsibilities
- **Door system**: Spawn, open, close, lock, check collisions; manage multi-door linkage; update `areaconnect` matrix when doors transition between open/closed states
- **Elevator system**: Manage two-location fast-travel with locked doors; handle player requests; teleport actors/statics between locations; optionally play elevator music
- **Push-wall system**: Spawn and update pushable/moving walls; track momentum and state; connect areas when walls move; handle collision resolution
- **Masked-wall system**: Manage transparent/shootable walls with animated frame sequences; handle break animations; maintain active/inactive linked lists
- **Touch-plate/trigger system**: Link actions (door open/close, push walls, lights, objects) to tile triggers; manage action queues and state callbacks
- **Area connectivity**: Maintain and update `areaconnect[][]` bidirectional connectivity matrix; compute `areabyplayer[]` via recursive traversal; determine visibility and sound propagation

## Key Types / Data Structures

| Name | Kind (struct/enum/class/typedef/interface/trait) | Purpose |
|------|-------|---------|
| `doorobj_t` | struct | Door with position, lock level, action state (open/closed/opening/closing), texture, sound handle, vertical orientation |
| `pwallobj_t` | struct | Push wall with position (x,y), momentum, direction, speed, texture, state machine (npushed/pushing/pushed/moving) |
| `maskedwallobj_t` | struct | Masked (transparent/shootable) wall with textures (top/mid/bottom), flags (shootable, blocking, etc.), linked-list ptrs |
| `animmaskedwallobj_t` | struct | Animation state for breakable masked walls; frame counter and tic countdown |
| `elevator_t` | struct | Elevator with source/dest tile coords, associated doors, state machine (ras/rad/mts/mtd/doorclosing), tic counter |
| `touchplatetype` | struct | Touch plate action: action/swapaction function pointers, target object, delay/trigger/done flags, linked-list ptrs |
| `estate` | enum | Elevator state: `ev_ras` (ready at source), `ev_rad` (ready at dest), `ev_mts` (moving to source), `ev_mtd` (moving to dest), `ev_doorclosing` |
| `masked_walls` | enum | Masked wall types (peephole, dogwall, multi-pane, normal, arch, platform, switch, gate, railing) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ELEVATOR[MAXELEVATORS]` | elevator_t[] | global | Array of all elevators in level |
| `_numelevators` | int | global | Count of active elevators |
| `doorobjlist[MAXDOORS]` | doorobj_t*[] | global | Array of pointers to all doors |
| `doornum` | int | global | Count of doors spawned |
| `pwallobjlist[MAXPWALLS]` | pwallobj_t*[] | global | Array of pointers to all push walls |
| `pwallnum` | int | global | Count of push walls |
| `maskobjlist[MAXMASKED]` | maskedwallobj_t*[] | global | Array of pointers to all masked walls |
| `maskednum` | int | global | Count of masked walls |
| `FIRSTMASKEDWALL`, `LASTMASKEDWALL` | maskedwallobj_t* | global | Doubly-linked list of active masked walls |
| `FIRSTANIMMASKEDWALL`, `LASTANIMMASKEDWALL` | animmaskedwallobj_t* | global | Doubly-linked list of animating masked walls |
| `touchplate[MAXTOUCHPLATES]` | touchplatetype*[] | global | Array of touch-plate action linked lists |
| `lastaction[MAXTOUCHPLATES]` | touchplatetype*[] | global | Tail pointers for each touch-plate list |
| `touchindices[MAPSIZE][MAPSIZE]` | byte[][] | global | Maps tile coordinates to touch-plate index+1 (0 if none) |
| `lasttouch` | byte | global | Highest touch-plate index used |
| `numactions[MAXTOUCHPLATES]` | byte[] | global | Action count per touch plate |
| `TRIGGER[MAXTOUCHPLATES]` | byte[] | global | Trigger state per touch plate |
| `areaconnect[NUMAREAS][NUMAREAS]` | byte[][] | global | Bidirectional connectivity; >0 = connected |
| `areabyplayer[NUMAREAS]` | boolean[] | global | Which areas are reachable from player's current area |
| `touchactions[NUMTOUCHPLATEACTIONS]` | function ptr[] | static | Lookup table mapping action index to function pointer |

## Key Functions / Methods

### SpawnDoor
- **Signature:** `void SpawnDoor(int tilex, int tiley, int lock, int texture)`
- **Purpose:** Create a new door at a map tile with given lock level and texture type; set up collision, sidepics, and adjacent door marking.
- **Inputs:** Tile position (x,y), lock level (0=unlocked, 1-4=key type, 5=elevator-linked), texture type (determines visuals)
- **Outputs/Return:** None; modifies global `doorobjlist[]`, `doornum`, `tilemap`, `actorat`
- **Side effects:** Allocates door struct from level heap; marks tilemap with door marker (0x8000 bit); pre-caches textures; increments `doornum`
- **Calls:** `Z_LevelMalloc`, `MAPSPOT`, `W_GetNumForName`, `IsDoor`, `IsWall`, `PreCacheLump`, `SD_PreCacheSoundGroup`
- **Notes:** Detects multi-door linkage (adjacent doors) by checking neighbors. Sets `DF_MULTI` flag if doors are horizontally or vertically adjacent. Vertical orientation auto-detected from wall layout.

### OpenDoor
- **Signature:** `void OpenDoor(int door)`
- **Purpose:** Begin door-opening animation by setting action state to `dr_opening`.
- **Inputs:** Door index
- **Outputs/Return:** None
- **Side effects:** Modifies `doorobjlist[door]->action` and `ticcount`
- **Calls:** None
- **Notes:** Resets open-time counter if already open; allows re-triggering.

### DoorOpening
- **Signature:** `void DoorOpening(int door)`
- **Purpose:** Update door position during opening animation; connect areas when door first starts opening.
- **Inputs:** Door index
- **Outputs/Return:** None; updates `doorobjlist[door]->position`, `texture`, `ticcount`, `action`
- **Side effects:** Increments `areaconnect[][]` matrix on first frame; calls `ConnectAreas()`; plays sound; transitions to `dr_open` when fully extended
- **Calls:** `ConnectAreas`, `SD_PlaySoundRTP`, `SD_PanRTP`
- **Notes:** Position is fixed-point (0xffff = fully open); texture index derived from position to animate opening frame. Area connection is crucial for sound and AI visibility propagation.

### DoorClosing
- **Signature:** `void DoorClosing(int door)`
- **Purpose:** Update door position during closing animation; disconnect areas when fully closed.
- **Inputs:** Door index
- **Outputs/Return:** None; updates position and action state
- **Side effects:** Decrements `areaconnect[][]` when position crosses halfway mark; plays door-hit sound; restores collision at destination
- **Calls:** `ResolveDoorSpace`, `ConnectAreas`, `SD_PlaySoundRTP`, `SD_PanRTP`
- **Notes:** Inverse of `DoorOpening`; also handles intermediate multi-door states.

### OperateDoor
- **Signature:** `void OperateDoor(int keys, int door, boolean localplayer)`
- **Purpose:** Handle player interaction with a locked door; check keys and permissions before allowing state change.
- **Inputs:** Bitmask of keys player owns, door index, whether this is local player (for UI messages)
- **Outputs/Return:** None
- **Side effects:** May call `UseDoor()` to toggle door state; plays reject sound if locked; adds message to HUD if local player and insufficient keys
- **Calls:** `UseDoor`, `SD_Play`, `AddMessage`
- **Notes:** Checks `DF_ELEVLOCKED` flag (prevents use if elevator is traveling) and gas-door barrier (`GASVALUE`). Elevator-linked doors have lock=5.

### UseDoor
- **Signature:** `void UseDoor(int door)`
- **Purpose:** Toggle door state: open→close or close→open, respecting current animation state.
- **Inputs:** Door index
- **Outputs/Return:** None
- **Side effects:** Calls `OpenDoor` or `CloseDoor`; stops current sound if transitioning from mid-animation
- **Calls:** `UtilizeDoor`, `DoorReadyToClose`, `SD_StopSound`, `OpenDoor`, `CloseDoor`
- **Notes:** Handles multi-doors via `UtilizeDoor` so all linked doors transition together.

### UtilizeDoor
- **Signature:** `void UtilizeDoor(int door, void (*action)(int))`
- **Purpose:** Apply an action (open/close) to a door and all adjacent multi-doors in the same line.
- **Inputs:** Door index, action function pointer (e.g., `OpenDoor`, `CloseDoor`)
- **Outputs/Return:** None
- **Side effects:** Calls action on primary door and all multi-doors in vertical or horizontal direction
- **Calls:** Action function on multiple doors
- **Notes:** Traverses `tilemap` in both +/- directions from primary door until non-multi door or edge found.

### DoorReadyToClose
- **Signature:** `boolean DoorReadyToClose(int door)`
- **Purpose:** Check if door and all adjacent multi-doors have no actors blocking them.
- **Inputs:** Door index
- **Outputs/Return:** `true` if safe to close, `false` if collision detected
- **Side effects:** None
- **Calls:** `DoorUnBlocked`, `M_ISDOOR` macro
- **Notes:** Used before initiating close to prevent crushing actors. Checks relative positions with `MINDIST` offset.

### SpawnPushWall
- **Signature:** `void SpawnPushWall(int tilex, int tiley, int lock, int texture, int dir, int type)`
- **Purpose:** Create a push wall at a map tile; set up momentum, speed, and initial state.
- **Inputs:** Tile (x,y), lock status, texture, push direction, type (0-3: push, 1-2: fast push, 3-4: move)
- **Outputs/Return:** None; updates global push-wall list
- **Side effects:** Allocates push-wall struct; sets `actorat[x][y]`; marks `tilemap`; pre-caches texture; increments `pwallnum`; may increment `gamestate.secrettotal`
- **Calls:** `Z_LevelMalloc`, `GetAreaNumber`, `ActivateMoveWall`, `SD_PreCacheSoundGroup`
- **Notes:** Types 3-4 are auto-moving (no player push needed). Secret count only incremented for non-automoving walls on first load.

### WallPushing
- **Signature:** `void WallPushing(int pwall)`
- **Purpose:** Update push wall during push animation; handle tile transitions and collision.
- **Inputs:** Push-wall index
- **Outputs/Return:** None; updates position, momentum, state
- **Side effects:** Updates `tilemap`, `actorat`, `mapseen`; may call `ResolveDoorSpace` if collision; transitions to `pw_pushed` or resets to `pw_npushed` if directional marker found
- **Calls:** `PushWallMove`, `ResolveDoorSpace`, `ClearActorat`, `SetActorat`
- **Notes:** Uses fixed-point momentum; state counter tracks push progress. If wall reaches marker tile with direction arrow, can continue or finish based on lock status.

### MovePWalls
- **Signature:** `void MovePWalls(void)`
- **Purpose:** Main update loop for all push walls; called once per frame from game loop.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates all `pw_pushing` and `pw_moving` walls; pans sound
- **Calls:** `WallPushing`, `WallMoving`, `SD_PanRTP`
- **Notes:** Called from `PlayLoop` context.

### ProcessElevators
- **Signature:** `void ProcessElevators(void)`
- **Purpose:** Main elevator state machine; handle transitions and execute arrival/departure actions.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates elevator state, tic counters, door states, music
- **Calls:** `ExecuteElevatorStopActions`, `CheckElevatorStart`
- **Notes:** Runs once per frame. Elevator states are: `ev_ras`/`ev_rad` (idle at source/dest), `ev_mts`/`ev_mtd` (in transit), `ev_doorclosing` (door closing before transit).

### Teleport
- **Signature:** `void Teleport(elevator_t *eptr, int destination)`
- **Purpose:** Instantaneously move all actors/statics at source tile to destination tile.
- **Inputs:** Elevator pointer, destination flag (0=src→dst, 1=dst→src)
- **Outputs/Return:** None
- **Side effects:** Updates x,y,tilex,tiley of all actors/statics in source tile; updates area numbers and active lists; screen shake if player teleported
- **Calls:** Actor/static linked-list iteration; `RemoveFromArea`, `MakeLastInArea`
- **Notes:** Handles both actors and statics; re-links them into area-specific lists.

### SpawnMaskedWall
- **Signature:** `void SpawnMaskedWall(int tilex, int tiley, int which, int flags)`
- **Purpose:** Create a masked (partially transparent/shootable) wall; set up textures and area connectivity.
- **Inputs:** Tile (x,y), wall type enum (peephole, dogwall, multi-pane, arch, platform, etc.), flags (shootable, blocking, etc.)
- **Outputs/Return:** None; updates global masked-wall list
- **Side effects:** Allocates struct; marks `tilemap` with masked-wall marker; may connect or disconnect areas; pre-caches textures
- **Calls:** `Z_LevelMalloc`, `W_GetNumForName`, `IsMaskedWall`, `IsWall`, `PreCacheLump`, `SD_PreCacheSound`
- **Notes:** Platform walls (type 14-20) have special handling for metal vs. non-metal with different animated textures. Many types are commented out for shareware builds.

### UpdateMaskedWall
- **Signature:** `int UpdateMaskedWall(int num)`
- **Purpose:** Check if masked wall has been shot/broken; spawn animation and handle multi-wall breaking.
- **Inputs:** Masked-wall index
- **Outputs/Return:** 1 if wall was shot, 0 otherwise
- **Side effects:** Calls `SpawnAnimatedMaskedWall` if `MW_SHOOTABLE` flag set; clears flag; plays sound; handles multi-wall groups
- **Calls:** `CheckMaskedWall`, `SpawnAnimatedMaskedWall`, `SD_PlaySoundRTP`
- **Notes:** If `MW_MULTI` flag set, also checks adjacent multi-walls in same direction.

### DoAnimatedMaskedWalls
- **Signature:** `void DoAnimatedMaskedWalls(void)`
- **Purpose:** Update all animated masked wall frame counters; remove completed animations.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates `maskobjlist[].bottomtexture` for animation frames; calls `KillAnimatedMaskedWall` when complete
- **Calls:** `KillAnimatedMaskedWall`
- **Notes:** Called each frame. Tic counts down; when zero, advance frame and reset tic counter.

### TriggerStuff
- **Signature:** `void TriggerStuff(void)`
- **Purpose:** Main touch-plate processing loop; check which plates are triggered and execute queued actions.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Executes action callbacks via function pointers stored in touch plates; swaps action and swapaction; updates triggered/done flags; displays messages on baby difficulty
- **Calls:** Action functions (e.g., `OpenDoor`, `ActivatePushWall`, `EnableObject`, `ActivateLight`); `DisplayMessageForAction`
- **Notes:** Player position checked against `touchindices[][]`. Actions fire when tic countdown reaches zero. Swapaction fires on next trigger (toggle behavior).

### Link_To_Touchplate
- **Signature:** `void Link_To_Touchplate(word touchlocx, word touchlocy, void (*maction)(int), void (*swapaction)(int), int wobj, int delaytime)`
- **Purpose:** Link an action callback to a touch-plate tile with optional delay.
- **Inputs:** Touch-plate tile (x,y), action and swap-action function pointers, target object index, delay in tics
- **Outputs/Return:** None
- **Side effects:** Allocates and links action node; increments `numactions[]` and `totalactions`
- **Calls:** `Z_LevelMalloc`, `AddTouchplateAction`
- **Notes:** Used by map loader to bind doors/walls/objects to touch plates.

### SaveDoors / LoadDoors
- **Signature:** `void SaveDoors(byte ** buf, int * size)` / `void LoadDoors(byte * buf, int size)`
- **Purpose:** Serialize/deserialize door states (position, action, lock, ticcount, elevator index) for save-game persistence.
- **Inputs/Outputs:** Buffer pointer and size (output for save; input for load)
- **Side effects:** Allocates buffer on save; restores door state and area connectivity on load; calls `FixDoorAreaNumbers`
- **Calls:** `SafeMalloc`, `SafeFree`, `SetupDoors`, `FixDoorAreaNumbers`, memcpy
- **Notes:** Encodes position+action in single int. Restores area connection by calling `DoorOpening` if door not fully closed.

### SavePushWalls / LoadPushWalls
- **Signature:** Similar to SaveDoors/LoadDoors
- **Purpose:** Persist push-wall state (position, momentum, direction, action, speed).
- **Inputs/Outputs:** Buffer pointer and size
- **Side effects:** Restores push-wall collision data; may call `ConnectPushWall` if position changed
- **Calls:** `SetupPushWalls`, `Z_LevelMalloc`, `ConnectPushWall`, `SetActorat`, `FinishPushWall`, `ResetPushWall`
- **Notes:** Handles state transitions (npushed→pushing vs. pushing→moved).

### SaveMaskedWalls / LoadMaskedWalls
- **Signature:** Similar pattern
- **Purpose:** Persist masked-wall animation/shootable state (flags only).
- **Inputs/Outputs:** Buffer pointer and size
- **Side effects:** Calls `UpdateMaskedWall` if flags have changed; updates texture if `MW_SWITCHON` flag set
- **Calls:** `SetupMaskedWalls`, `FixMaskedWallAreaNumbers`, `UpdateMaskedWall`

### SaveTouchPlates / LoadTouchPlates
- **Signature:** Similar pattern
- **Purpose:** Persist touch-plate action state (triggered, ticcount, action indices, target objects).
- **Inputs/Outputs:** Buffer pointer and size
- **Side effects:** Decodes actor/stat pointers from saved indices using `objlist` and `GetStatForIndex`
- **Calls:** `Z_LevelMalloc`, `GetIndexForAction`, `GetStatForIndex`, `AddTouchplateAction`, memcpy
- **Notes:** Complex serialization: converts actor/stat pointers to/from indices with flags (`FL_TACT`, `FL_TSTAT`). Conditional debug output if `LOADSAVETEST == 1`.

### ConnectAreas
- **Signature:** `void ConnectAreas(void)`
- **Purpose:** Update which areas are accessible from player position(s) via recursive traversal of connectivity matrix; activate/deactivate actors, stats, masked walls accordingly.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets `areabyplayer[]`; sets `FL_ABP` flag on actors/statics in reachable areas; calls `MakeActive/MakeInactive` and `MakeMaskedWallActive/Inactive`; disables master disk objects
- **Calls:** `RecursiveConnect`, `MakeActive`, `MakeInactive`, `MakeStatActive`, `MakeStatInactive`, `MakeMaskedWallActive`, `MakeMaskedWallInactive`
- **Notes:** Called whenever area connectivity changes (door open/close, push wall move). Essential for performance (only process visible areas) and audio (sounds travel between connected areas).

### CheckTile
- **Signature:** `boolean CheckTile(int x, int y)`
- **Purpose:** Determine if a tile is walkable (no walls, doors, actors, items, platforms, windows, invalid area).
- **Inputs:** Tile coordinates (x,y)
- **Outputs/Return:** `true` if empty and valid, `false` otherwise
- **Side effects:** None
- **Calls:** `actorat[][]` and `tilemap[][]` lookups; `M_ISACTOR`, `IsPlatform`, `IsWindow`, `AREANUMBER`
- **Notes:** Used for spawn-point selection and pathfinding validation. Excludes setup-game mode from certain checks.

### FindEmptyTile
- **Signature:** `void FindEmptyTile(int *stilex, int *stiley)`
- **Purpose:** Find nearest walkable tile within an area, spiraling outward from a starting position.
- **Inputs/Outputs:** Pointer to tile (x,y); modified in place with valid tile
- **Side effects:** Modifies input tile coords
- **Calls:** `CheckTile`, `AREANUMBER`
- **Notes:** Expands search in expanding square pattern. Used for actor spawning when initial tile is blocked.

## Control Flow Notes

**Initialization:**
- `InitDoorList()` called at level load; zeros all global arrays, initializes counters
- `InitAreas()` zeros area connectivity
- `InitElevators()` zeros elevator array
- Doors/walls/plates spawned during map parsing
- `FixDoorAreaNumbers()` and `FixMaskedWallAreaNumbers()` correct area assignments after all tiles loaded

**Main Game Loop (each frame):**
1. `MoveDoors()` – update door animations
2. `MovePWalls()` – update push wall animations  
3. `ProcessElevators()` – handle elevator state machine
4. `TriggerStuff()` – process touch plates and fire actions
5. `DoAnimatedMaskedWalls()` – update breakable wall animations
6. `ConnectAreas()` – (called when door/wall state changes) recompute reachable areas

**Player Interaction:**
- Player presses use key on door → `OperateDoor()` → `UseDoor()` → `UtilizeDoor()`
- Player presses use key on push wall → `OperatePushWall()` → `SetupPushWall()`
- Player steps on touch plate → next `TriggerStuff()` loop processes queued actions

**Save/Load:**
- At game save: `SaveDoors/SavePushWalls/SaveMaskedWalls/SaveTouchPlates/SaveElevators()` called
- At game load: corresponding `Load*()` functions restore state

## External Dependencies

**Notable includes / imports:**
- `rt_def.h` – core constants (MAPSIZE, NUMAREAS, direction enums, flags)
- `rt_sound.h` – sound system (SD_Play, SD_PlaySoundRTP, SD_PanRTP, MU_StartSong, MU_RestoreSongPosition)
- `rt_actor.h` – actor types and macros (M_ISACTOR, objtype, classtype, actor lists)
- `rt_stat.h` – static object types (statobj_t, stat_t enum)
- `rt_ted.h` – tile editor data structures
- `z_zone.h` – memory allocation (Z_LevelMalloc, Z_Free)
- `w_wad.h` – WAD file access (W_GetNumForName, W_CacheLumpNum)
- `rt_draw.h` – rendering hints
- `rt_main.h` – global game state (gamestate, tics, loadedgame, insetupgame)
- `rt_playr.h` – player object type (player, PLAYER[])
- `rt_util.h` – utilities
- `rt_menu.h` – HUD messages (AddMessage)
- `rt_msg.h` – message types
- `rt_game.h` – game state
- `rt_vid.h` – video I/O
- `rt_net.h` – network/multiplayer
- `isr.h` – interrupts
- `develop.h` – debug/development flags
- `rt_rand.h` – random number generation
- `engine.h` – 3D engine
- `stdlib.h`, `string.h` – C standard library (memcpy, memset)
- `memcheck.h` – memory debugging

**Defined elsewhere (not in this file):**
- `actorat[][]`, `sprites[][]`, `tilemap[][]`, `mapplanes[][]` – global map data
- `firstactive`, `firstactivestat`, `FIRSTSTAT`, `LASTSTAT` – object linked lists
- `PLAYER[]`, `numplayers` – player array
- `gamestate`, `MISCVARS` – global game state structs
- `spotvis[][]`, `mapseen[][]` – visibility tracking
- `walls[]`, `GetWallIndex()` – wall texture database
- `objlist[]` – actor pointer array (for save/load)
- `Clocks[]`, `numclocks` – clock/timer system (referenced but not defined here)
- `demoplayback`, `demorecord` – demo flags
- `loadedgame`, `insetupgame` – state flags
- `SHAKETICS` – screen shake counter
- `firstactivestat`, `lastactivestat` – stat active list
- `BATTLE_CheckGameStatus()`, `BATTLEMODE` – battle mode system
- Sound functions: `SD_StopSound()`, `MU_StoreSongPosition()`, `MusicStarted()`, `GameRandomNumber()`
- Other: `Error()`, `SoftError()`, `Debug()` – diagnostics
- Rendering: `VL_DecompressLBM()`, `VW_UpdateScreen()`, `I_Delay()`
