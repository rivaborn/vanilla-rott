# rott/rt_stat.h

## File Purpose
Header for the static objects (sprites, decorations, items, hazards) subsystem. Defines structures and functions to spawn, manage, animate, and save/load static game entities placed in levels—everything from weapons and powerups to environmental hazards and decorative objects.

## Core Responsibilities
- Define static object type enumeration (`stat_t`) and metadata structures
- Manage linked lists of active/inactive static objects
- Provide spawning and initialization functions for various object types
- Handle state transitions (activate/deactivate) and spatial management
- Implement animated wall updates and sprite frame logic
- Support save/restore of static state and respawn queues
- Manage switches and lighting state

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `stat_t` | enum | 91 static object types (lights, items, weapons, decorations, hazards, etc.) |
| `statobj_t` | struct | Runtime instance of a static object; contains position, rendering, state, and doubly-linked list pointers for active/inactive queues |
| `animwall_t` | struct | Active animated wall with frame counter and texture state |
| `respawn_t` | struct | Respawn point metadata (position, type, spawn delay, linked object) |
| `statinfo` | struct | Template/metadata for each stat type (sprite, hp, damage, animation count, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `firstactivestat`, `lastactivestat` | `statobj_t*` | extern | Head/tail pointers of doubly-linked active objects list |
| `firstemptystat`, `lastemptystat` | `statobj_t*` | extern | Head/tail pointers of free object pool |
| `FIRSTSTAT`, `LASTSTAT` | `statobj_t*` | extern | Global bounds of static object array |
| `sprites[MAPSIZE][MAPSIZE]` | `statobj_t*[][]` | extern | 2D grid of stat objects by tile position for spatial lookups |
| `stats[NUMSTATS]` | `statinfo[]` | extern | Metadata templates for each of 91 stat types |
| `switches[MAXSWITCHES]` | `wall_t[]` | extern | Lever/button switch definitions (64 max) |
| `firstrespawn`, `lastrespawn` | `respawn_t*` | extern | Linked list of active respawn queues |
| `animwalls[MAXANIMWALLS]` | `animwall_t[]` | extern | Animated wall instances (17 max) |
| `statcount` | int | extern | Total active static objects |
| `diagonal[9][9]`, `opposite[9]` | `dirtype[][]`, `dirtype[]` | extern | Direction lookup tables |

## Key Functions / Methods

### InitStaticList
- **Purpose:** Initialize the free pool of static object structures at startup.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets up `firstemptystat`/`lastemptystat` linked list.
- **Calls:** (implementation not in this file)
- **Notes:** Called once during engine initialization.

### InitAnimatedWallList
- **Purpose:** Prepare the animated wall array and set initial state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes `animwalls[]` array.
- **Calls:** (implementation not in this file)

### SpawnStatic
- **Purpose:** Create and place a static object instance in the game world at a map grid position.
- **Inputs:** `tilex`, `tiley` (map tile coords), `mtype` (stat_t enum), `zoffset` (height adjustment)
- **Outputs/Return:** None
- **Side effects:** Allocates from free pool, inserts into `sprites[tilex][tiley]` grid, initializes position/state.
- **Calls:** (defined in implementation file)

### SpawnSolidStatic
- **Purpose:** Place a solid/collision-enabled static object and register its spatial bounds.
- **Inputs:** `temp` (pre-populated `statobj_t*`)
- **Outputs/Return:** None
- **Side effects:** Adds object to collision/spatial structures.
- **Calls:** (defined in implementation)

### SpawnInertStatic
- **Purpose:** Spawn a non-interactive static object (decoration, rubble, gibs) at arbitrary 3D position.
- **Inputs:** `x`, `y`, `z` (world coords), `mtype` (stat_t)
- **Outputs/Return:** None
- **Side effects:** Creates statobj with fixed coordinates, likely added to rendering queue.
- **Calls:** (defined in implementation)

### MakeStatActive / MakeStatInactive
- **Purpose:** Move a static object between active/inactive doubly-linked lists.
- **Inputs:** `statobj_t*` (object to transition)
- **Outputs/Return:** None
- **Side effects:** Unlinks from current list, relinks to target list; `statcount` updated.
- **Calls:** (linked-list manipulation in implementation)

### AddStatic / RemoveStatic
- **Purpose:** Insert/remove a static object from the world and its spatial data structures.
- **Inputs:** `statobj_t*`
- **Outputs/Return:** None
- **Side effects:** Updates `sprites[][]` grid, free pool, and active lists.
- **Calls:** `MakeStatActive`/`MakeStatInactive`, (spatial updates)

### ActivateLight / DeactivateLight
- **Purpose:** Enable/disable dynamic lighting for light-emitting static objects.
- **Inputs:** `int` (stat index or light ID)
- **Outputs/Return:** None
- **Side effects:** Updates global light state.
- **Calls:** (lighting subsystem in implementation)

### TurnOnLight / TurnOffLight
- **Purpose:** Immediately switch a light's on/off state with optional color/intensity.
- **Inputs:** `int` (light ID), `int` (parameter, likely color/intensity)
- **Outputs/Return:** None
- **Side effects:** Updates light rendering state.
- **Calls:** (lighting subsystem)

### AnimateWalls
- **Purpose:** Update animated wall textures each frame (increment frame counters, wrap textures).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies `animwalls[].ticcount` and `.texture` for each active animation.
- **Calls:** (texture management in implementation)

### DoSprites
- **Purpose:** Main per-frame update loop for all active static objects (animation, state ticks, lifetime).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Decrements `ticcount` on active objects, handles expiration/respawn logic.
- **Calls:** Animation, respawn, and removal functions.
- **Notes:** Called once per game frame.

### SaveStatics / LoadStatics
- **Purpose:** Serialize/deserialize all active and respawning static objects for savegame.
- **Inputs:** `byte**` buffer pointer, `int*` size
- **Outputs/Return:** None
- **Side effects:** Writes/reads state from buffer; manages memory allocation.
- **Calls:** (serialization in implementation)

### SaveAnimWalls / LoadAnimWalls, SaveSwitches / LoadSwitches
- **Purpose:** Persist animated wall and switch state across save/load cycles.
- **Inputs/Outputs:** Similar to SaveStatics/LoadStatics.
- **Side effects:** Writes/reconstructs animation and switch state.

### SpawnStaticDamage
- **Purpose:** Spawn gibs or particle effects when a static object is destroyed.
- **Inputs:** `statobj_t*` (destroyed object), `int` (damage direction/angle)
- **Outputs/Return:** None
- **Side effects:** Creates multiple inert statics (gibs/fragments) around impact point.
- **Calls:** `SpawnInertStatic`

### Set_NewZ_to_MapValue
- **Purpose:** Adjust a static object's Z coordinate to match floor height at its XY position.
- **Inputs:** `fixed*` (Z pointer), `int` (map tile), `const char*` (debug context), `int`, `int` (tile coords)
- **Outputs/Return:** None (modifies Z in-place)
- **Side effects:** Updates object's Z position.
- **Calls:** (terrain/elevation lookup in implementation)

### RemoveFromFreeStaticList / CheckCriticalStatics
- **Purpose:** Manage object pool; check/validate critical static objects.
- **Inputs:** `statobj_t*` for RemoveFromFreeStaticList; none for CheckCriticalStatics
- **Outputs/Return:** None
- **Side effects:** Pool/validation updates.

---

**Notes on trivial functions:**
- `SetupAnimatedWall(int)`: Setup a single animated wall definition.
- `SpawnSwitchThingy(int, int)`: Spawn a switch object at grid position.

## Control Flow Notes
**Initialization phase** → `InitStaticList()` and `InitAnimatedWallList()` set up empty pools.
**Level setup** → `SpawnStatic()` and related functions populate the world from map data.
**Main game loop** → `DoSprites()` and `AnimateWalls()` execute once per frame to update all active objects, handle animations, lifecycle, and respawns.
**Shutdown/Save** → `SaveStatics()`, `SaveAnimWalls()`, `SaveSwitches()` serialize state before level exit or savegame.

## External Dependencies
- **Includes:** `rt_ted.h` (map structures, wall definitions, spawn locations, lighting)
- **Defined elsewhere:**
  - `thingtype` (entity type enum, likely from a core types header)
  - `fixed` (fixed-point number type, likely from math headers)
  - `byte`, `word`, `signed char` (primitive type defs)
  - `dirtype` (direction enumeration)
  - `MAPSPOT()`, `MAPSIZE`, `NUMAREAS` (map access macros/constants)
  - `VBLCOUNTER`, `MAXPLAYERS` (timing/game constants)
  - Lighting subsystem (called by `ActivateLight`, `TurnOnLight`, etc.)
