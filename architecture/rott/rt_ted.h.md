# rott/rt_ted.h

## File Purpose
Header for Ted (level editor) integration, declaring map/level loading, entity setup, and game initialization. Defines data structures for levels, walls, teams, spawn points, and clocks; exports functions to load maps, configure game entities (players, doors, walls, switches, lights), and manage level resources.

## Core Responsibilities
- Define map file formats and in-memory level representations (RTLMAP, mapinfo_t)
- Export map loading and level initialization pipeline
- Manage spawn locations, team assignments, and player setup
- Configure map entities: walls, doors, switches, clocks, lights, animated walls
- Provide asset precaching and map metadata lookup
- Handle special level features (exits, platforms, push walls, links)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `teamtype` | struct | Team data: member count, uniform color, spawn location (tilex/tiley), direction |
| `wall_t` | struct | Wall tile: entity type, flags (switch/on/reversible/damage), HP, tile ID, position |
| `mapinfo_t` | struct | Map metadata: map number, level name (23 chars) |
| `mapfileinfo_t` | struct | Container for multiple maps (array of 100 mapinfo_t) |
| `RTLMAP` | struct | Level map header: CRC, RLE tag, start/length of 3 planes, map name, special flags |
| `_2dvec` | struct | 2D vector with direction: x, y, dir |
| `str_clock` | struct | Clock/timer entity: two time values, target tile position, link index |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TEAM[MAXPLAYERS]` | teamtype[] | extern | Team data for all players (up to 5 or 11) |
| `SPAWNLOC[MAXSPAWNLOCATIONS]` | _2dvec[] | extern | Spawn points (max 50) |
| `FIRST, SECOND` | _2dvec | extern | Notable spawn locations |
| `walls[MAXWALLTILES]` | wall_t[] | extern | Wall tile array |
| `Clocks[MAXCLOCKS]` | str_clock[] | extern | Timer/clock entities (max 10) |
| `mapplanes[3]` | unsigned short int*[] | extern | Three planes of map tile data |
| `numclocks` | int | extern | Active clock count |
| `NUMSPAWNLOCATIONS` | int | extern | Active spawn location count |
| `mapwidth, mapheight` | int | extern | Current level dimensions |
| `LevelName[80]` | char[] | extern | Current level name string |
| `ISRTL` | boolean | extern | RTL format indicator |
| `insetupgame` | boolean | extern | Level setup in progress flag |
| `fog, lightsource` | int | extern | Lighting parameters |
| `SNAKELEVEL` | int | extern | Snake level identifier |
| `ELEVATORLOCATION` | word | extern | Elevator tile reference |

## Key Functions / Methods

### LoadTedMap
- **Signature:** `void LoadTedMap(const char *extension, int mapnum)`
- **Purpose:** Load map data from disk file based map number and file extension
- **Inputs:** File extension (e.g., "MAP"), map number index
- **Outputs/Return:** None (modifies global mapplanes, mapwidth, mapheight)
- **Side effects:** Allocates/loads map plane data, reads level CRC and metadata
- **Calls:** File I/O functions (not visible in this header)

### SetupGameLevel
- **Signature:** `void SetupGameLevel(void)`
- **Purpose:** Full game level initialization pipeline
- **Inputs:** None (uses global map data)
- **Outputs/Return:** None (initializes all level entities and systems)
- **Side effects:** Calls SetupWalls, SetupDoors, SetupClocks, SetupPlayers, precaches assets
- **Notes:** Called after LoadTedMap; idempotent within game session

### ScanInfoPlane
- **Signature:** `void ScanInfoPlane(void)`
- **Purpose:** Parse info plane (map metadata layer) to identify clocks, links, doors, lights
- **Inputs:** None (uses global mapplanes)
- **Outputs/Return:** None (populates Clocks[], door/switch structures)

### SetupWalls / SetupDoors / SetupClocks / SetupPlayers / SetupSwitches / SetupMaskedWalls / SetupPushWalls / SetupLights
- **Purpose:** Specialized entity setup functions called by SetupGameLevel
- **Inputs:** None (read global state)
- **Outputs/Return:** None (configure entity subsystems)

### PreCache / PreCacheGroup / PreCacheLump / PreCacheActor
- **Purpose:** Load and cache sprite/texture assets for the current level
- **Signature:** `void PreCache(void)`, `void PreCacheGroup(int, int)`, etc.
- **Inputs:** Asset IDs (lump numbers, actor/sprite indices), level ID
- **Outputs/Return:** None (loads from disk into memory)

### GetMapInfo / GetMapFileName / GetMapCRC
- **Purpose:** Map metadata lookup and retrieval
- **Signature:** `void GetMapInfo(mapfileinfo_t *mapinfo)`, `word GetMapCRC(int num)`, etc.
- **Inputs:** Map number or buffer pointer
- **Outputs/Return:** Map info struct (populated), CRC checksum, filename
- **Notes:** Read-only queries of map directory

### GetNextMap
- **Signature:** `int GetNextMap(int tilex, int tiley)`
- **Purpose:** Determine which level loads when player reaches tile (tilex, tiley), typically an exit
- **Inputs:** Tile coordinates
- **Outputs/Return:** Next map number (-1 if none)

### Illuminate / GetSongForLevel / CheckHolidays / IsChristmas
- **Purpose:** Utility functions: lighting update, music selection, holiday logic
- **Notes:** Likely called per-frame or per-level-load

### DoPanicMapping
- **Signature:** `boolean DoPanicMapping(void)`
- **Purpose:** Fallback/debug: generate/load emergency map if primary load fails
- **Outputs/Return:** Success flag

## Control Flow Notes
**Initialization phase:** `LoadTedMap()` → `ScanInfoPlane()` → `SetupGameLevel()` → `PreCache()` → `Illuminate()`.

**Per-frame:** Possibly `Illuminate()` if dynamic lighting. Clock/timer updates likely in main game loop (not exported here).

**Level transitions:** `GetNextMap()` used to determine destination; new level cycle begins.

**Shutdown:** Not visible in this header; likely cleanup in rt_ted.c implementation.

## External Dependencies
- **rottnet.h**: Provides `MAXPLAYERS`, `boolean`, `byte`, `word` typedefs; networking context
- **Implied:** thingtype (entity type enum), MAPSPOT macro, NUMAREAS constant, VBLCOUNTER (video sync), various subsystem functions (not declared here)
