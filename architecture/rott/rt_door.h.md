# rott/rt_door.h

## File Purpose
Header file for the ROTT game engine's door, wall, and elevator system. Defines data structures and function prototypes for interactive map objects including sliding doors, pushwalls, masked walls, elevators, and trigger plates.

## Core Responsibilities
- Define data structures for doors, pushwalls, elevators, masked walls, and touch-activated triggers
- Declare initialization functions for doors, elevators, and area connectivity
- Declare runtime update functions (door/wall/elevator movement, animation, state machine)
- Declare save/load functions for persistent state serialization
- Declare collision/area helper functions and query macros
- Define wall type flags and state enumerations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `doorobj_t` | struct | Door state: position, texture, lock status, animation frame, sound handle, open/closed state |
| `pwallobj_t` | struct | Pushwall state: position, momentum, animation frame, texture, lock and collision flags |
| `elevator_t` | struct | Elevator state machine: source/dest positions, door indices, state, door action queue, tic counter |
| `touchplatetype` | struct | Touch-activated trigger: action callbacks, linked-list pointers, trigger timer, completion state |
| `maskedwallobj_t` | struct | Masked/decorated wall: position, 3D textures (top/mid/bottom), flags, linked-list pointers |
| `animmaskedwallobj_t` | struct | Animation state for masked walls: animation frame counter, linked-list pointers |
| `estate` | enum | Elevator state: ready-at-source, ready-at-dest, moving-to-source, moving-to-dest, door-closing |
| `masked_walls` | enum | Wall decoration types: peephole, dogwall, platforms, gates, switches, railings, etc. |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ELEVATOR[MAXELEVATORS]` | `elevator_t[]` | global | Array of elevator state machines (max 16) |
| `doorobjlist[MAXDOORS]` | `doorobj_t*[]` | global | Pointers to active door objects (max 150) |
| `maskobjlist[MAXMASKED]` | `maskedwallobj_t*[]` | global | Pointers to active masked walls (max 300) |
| `pwallobjlist[MAXPWALLS]` | `pwallobj_t*[]` | global | Pointers to active pushwall objects (max 150) |
| `FIRSTMASKEDWALL`, `LASTMASKEDWALL` | `maskedwallobj_t*` | global | Doubly-linked list anchor pointers for masked walls |
| `FIRSTANIMMASKEDWALL`, `LASTANIMMASKEDWALL` | `animmaskedwallobj_t*` | global | Doubly-linked list anchor pointers for animated masked walls |
| `touchplate[MAXTOUCHPLATES]`, `lastaction[MAXTOUCHPLATES]` | `touchplatetype*[]` | global | Touch plate trigger array and state (max 64) |
| `TRIGGER[MAXTOUCHPLATES]` | `byte[]` | global | Trigger state flags for each touch plate |
| `doornum`, `maskednum`, `pwallnum`, `_numelevators` | `int` | global | Count of active objects in each list |
| `touchindices[MAPSIZE][MAPSIZE]`, `lasttouch` | `byte[][]`, `byte` | global | Map-indexed lookup: which touch plate at tile, and last-touched plate ID |
| `areaconnect[NUMAREAS][NUMAREAS]` | `byte[][]` | global | Area connectivity matrix (reachability between map areas) |
| `areabyplayer[NUMAREAS]` | `boolean[]` | global | Flag: whether each area has been visited by player |

## Key Functions / Methods

### SpawnDoor
- **Signature:** `void SpawnDoor(int, int, int, int)`
- **Purpose:** Instantiate a door at a map location with specified type and initial state
- **Inputs:** Tile coordinates (x, y), door type, initial state
- **Outputs/Return:** None; modifies global `doorobjlist` and `doornum`
- **Side effects:** Allocates door object, registers in global list, initializes textures and sound handle
- **Calls:** Not inferable from header
- **Notes:** Doors are indexed by position in `doorobjlist`; not by tile coordinate

### MoveDoors
- **Signature:** `void MoveDoors(void)`
- **Purpose:** Update animation frames and position for all active doors each frame
- **Inputs:** None (reads global `doorobjlist[]`)
- **Outputs/Return:** None
- **Side effects:** Updates door state machine, plays sounds, updates sound handle positions
- **Calls:** Not inferable from header
- **Notes:** Called once per frame; handles dr_opening and dr_closing states

### OperateDoor
- **Signature:** `void OperateDoor(int keys, int door, boolean localplayer)`
- **Purpose:** Toggle a door open or closed based on player input and lock status
- **Inputs:** Key flags, door index, local player flag (for multiplayer)
- **Outputs/Return:** None
- **Side effects:** Changes door state to opening/closing, may fail if locked
- **Calls:** Not inferable from header
- **Notes:** Respects lock and flag bits; localplayer flag likely affects network sync

### SpawnPushWall
- **Signature:** `void SpawnPushWall(int tilex, int tiley, int lock, int texture, int dir, int type)`
- **Purpose:** Instantiate a pushwall at a map location
- **Inputs:** Tile position, lock status, texture ID, direction, type
- **Outputs/Return:** None; registers in global `pwallobjlist`
- **Side effects:** Allocates pushwall object, initializes momentum and animation state
- **Calls:** Not inferable from header
- **Notes:** Pushwalls have locked/unlocked states and can move in four directions

### MovePWalls
- **Signature:** `void MovePWalls(void)`
- **Purpose:** Update momentum and position for all active pushwalls each frame
- **Inputs:** None (reads global `pwallobjlist[]`)
- **Outputs/Return:** None
- **Side effects:** Applies momentum, updates animation frame, plays collision/movement sounds
- **Calls:** Not inferable from header
- **Notes:** Handles pw_pushing and pw_moving states; applies friction/deceleration

### ProcessElevators
- **Signature:** `void ProcessElevators(void)`
- **Purpose:** Update elevator state machine and linked door operations each frame
- **Inputs:** None (reads global `ELEVATOR[]`)
- **Outputs/Return:** None
- **Side effects:** Advances elevator state (ev_ras → ev_mtd, etc.), opens/closes linked doors, tic counter
- **Calls:** `OperateElevatorDoor` (likely)
- **Notes:** Elevator state controls door synchronization; nextaction queue handles multi-action sequences

### InitElevators
- **Signature:** `void InitElevators(void)`
- **Purpose:** Initialize all elevators to idle state at spawn locations
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `ELEVATOR[]` array, sets `_numelevators` to 0
- **Calls:** Not inferable from header
- **Notes:** Called during level load

### InitAreas
- **Signature:** `void InitAreas(void)`
- **Purpose:** Mark all areas as unexplored and initialize area connectivity
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Zeros `areabyplayer[]`, sets up `areaconnect[][]` matrix
- **Calls:** `ConnectAreas` (likely)
- **Notes:** Part of level initialization pipeline

### ConnectAreas
- **Signature:** `void ConnectAreas(void)`
- **Purpose:** Compute which areas are reachable from which (flood-fill connectivity)
- **Inputs:** None (reads `tilemap`, door states)
- **Outputs/Return:** None
- **Side effects:** Populates `areaconnect[][]` matrix
- **Calls:** `RecursiveConnect` (likely)
- **Notes:** Used for AI pathfinding and visibility culling

### RecursiveConnect
- **Signature:** `void RecursiveConnect(int)`
- **Purpose:** Recursively mark all reachable areas from a starting area
- **Inputs:** Starting area number
- **Outputs/Return:** None
- **Side effects:** Updates `areaconnect[start][*]`
- **Calls:** Recursively calls itself
- **Notes:** Depth-first search from one area to all connected neighbors

### TriggerStuff
- **Signature:** `void TriggerStuff(void)`
- **Purpose:** Process all active touch plate triggers and decrement timers
- **Inputs:** None (reads global `touchplate[]`)
- **Outputs/Return:** None
- **Side effects:** Decrements `ticcount`, invokes action callbacks when triggered, clears completed plates
- **Calls:** User-supplied action callbacks in `touchplatetype`
- **Notes:** Called once per frame; supports delayed actions and clock-based timing

### Link_To_Touchplate
- **Signature:** `void Link_To_Touchplate(word, word, void (*)(int), void (*)(int), int, int)`
- **Purpose:** Register a touch plate trigger at a map location with action callbacks
- **Inputs:** Tile coordinates, action callback, swap callback, object index, timing parameters
- **Outputs/Return:** None
- **Side effects:** Allocates touchplate, registers in global `touchplate[]` and `touchindices[][]`
- **Calls:** Not inferable from header
- **Notes:** Two callbacks allow bidirectional triggers (on/off)

### SpawnMaskedWall
- **Signature:** `void SpawnMaskedWall(int tilex, int tiley, int which, int flags)`
- **Purpose:** Instantiate a masked/decorated wall (3D overlay) at a map location
- **Inputs:** Tile position, wall type (masked_walls enum), flags
- **Outputs/Return:** None
- **Side effects:** Allocates masked wall object, inserts into linked list
- **Calls:** Not inferable from header
- **Notes:** Masked walls are drawn after floor/ceiling for visual depth

### UpdateMaskedWall
- **Signature:** `int UpdateMaskedWall(int num)`
- **Purpose:** Update animation frame for a masked wall and return next frame index
- **Inputs:** Masked wall index/ID
- **Outputs/Return:** Next animation frame index
- **Side effects:** Increments animation counter, wraps to loop frame
- **Calls:** Not inferable from header
- **Notes:** Called during render phase for each animated masked wall

### SaveDoors, SavePushWalls, SaveMaskedWalls, SaveElevators
- **Signature:** `void Save*(byte ** buf, int * size)` (variants)
- **Purpose:** Serialize door/pushwall/wall/elevator state to a byte buffer for save game
- **Inputs:** Buffer pointer (output), size pointer (output)
- **Outputs/Return:** None; fills buffer and updates size
- **Side effects:** Allocates/writes to buffer
- **Calls:** Not inferable from header
- **Notes:** Paired with Load* functions for round-trip persistence

### LoadDoors, LoadPushWalls, LoadMaskedWalls, LoadElevators
- **Signature:** `void Load*(byte * buf, int size)` (variants)
- **Purpose:** Deserialize door/pushwall/wall/elevator state from a save game buffer
- **Inputs:** Buffer, buffer size
- **Outputs/Return:** None; restores global state
- **Side effects:** Repopulates global lists and linked lists
- **Calls:** Not inferable from header
- **Notes:** Inverse of Save* functions

**Notes on trivial functions:**
- `OpenDoor`, `CloseDoor`, `DoorOpen`, `DoorOpening`, `DoorClosing`: Likely simple state-setter functions or callbacks for door state machine
- `ActivatePushWall`, `ActivateMoveWall`, `WallPushing`, `PushWall`: Pushwall action helpers
- `ActivateAllPushWalls`, `CheckTile`, `FindEmptyTile`, `Number_of_Empty_Tiles_In_Area_Around`: Utility query functions
- `PlatformHeight`: Query function (likely for collision/rendering)
- `IsWall`, `IsDoor`, `IsMaskedWall`: Tilemap query macros/functions
- `FixDoorAreaNumbers`, `FixMaskedWallAreaNumbers`: Post-load correction helpers
- `DeactivateAnimMaskedWall`, `ActivateAnimMaskedWall`, `SpawnAnimatedMaskedWall`, `KillAnimatedMaskedWall`, `DoAnimatedMaskedWalls`: Masked wall animation lifecycle
- `MakeWideDoorVisible`, `LinkedCloseDoor`, `LinkedOpenDoor`: Door visibility/linking helpers
- `ClockLink`: Register time-based action callback

## Control Flow Notes

**Initialization phase (level load):**
- `InitDoorList()` → creates door objects and registers them
- `InitElevators()` → zero elevator state
- `InitAreas()` → clear visited flags
- `ConnectAreas()` → compute reachability matrix via `RecursiveConnect`
- `FixDoorAreaNumbers()`, `FixMaskedWallAreaNumbers()` → correct area tags post-load

**Frame/Update phase (per-tick):**
- `MoveDoors()` → advance door animation and state machine
- `MovePWalls()` → advance pushwall momentum and position
- `ProcessElevators()` → advance elevator state machine and trigger linked door actions
- `DoAnimatedMaskedWalls()` → advance masked wall animation frames
- `TriggerStuff()` → decrement touch plate timers and fire callbacks

**Player interaction:**
- `OperateDoor(keys, door, localplayer)` → player tries to open/close a door
- `OperatePushWall(pwall, dir, localplayer)` → player tries to push a wall
- Touch plates automatically trigger when player enters a tile, invoking registered callbacks

**Area system:**
- `areaconnect[][]` enables visibility culling and AI pathfinding
- `areabyplayer[]` tracks visited areas for level progression logic

## External Dependencies

- **Notable includes / imports:** None explicit in header; likely includes C standard library, game types (`thingtype`), and map headers
- **Defined elsewhere:**
  - `thingtype`: Likely enum or typedef for game object types
  - `tilemap[x][y]`: Global 2D array representing the map grid (bits encode door, wall, door-type info)
  - `NUMAREAS`, `MAPSIZE`: Preprocessor constants defining map dimensions
  - Sound system (`int soundhandle`): Likely external audio API
  - Texture system (`word texture`, `word alttexture`, `int sidepic`): Likely external rendering API
  - `tiling`, `area`, and collision geometry: Likely defined in map/collision modules
