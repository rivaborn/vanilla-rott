# rott/rt_ted.c

## File Purpose
Handles ROTT level/map file loading, precaching of level resources, and comprehensive level initialization including spawning actors, setting up doors, walls, and other interactive elements. Acts as the primary level setup orchestrator.

## Core Responsibilities
- Load map data from ROTT and Ted format files with RLEZ decompression
- Manage precache system for graphics, sounds, and other resources
- Initialize all tile-based entities (walls, doors, switches, animated walls)
- Spawn player start positions and multiplayer team locations
- Configure actors (enemies), static objects (pickups, hazards), and special geometry
- Handle map-specific features (elevators, platforms, lights, masked walls)
- Support version checking, game mode conversions (shareware/registered/low-memory)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `RTLMAP` | struct | Map file header with plane offsets, dimensions, metadata, CRC |
| `wall_t` | struct | Wall tile definition with flags, health, tile ID |
| `teamtype` | struct | Multiplayer team info: member count, color, spawn location |
| `str_clock` | struct | Timed door/switch definition with activation/deactivation times |
| `_2dvec` | struct | 2D integer position vector (x, y, dir) for spawn locations |
| `mapfileinfo_t` | struct | Container for array of map metadata from a map file |
| `cachetype` | struct | Precache entry with lump ID and priority level |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `mapplanes[3]` | `word*[3]` | global | Three map layers: background geometry, icon/actor layer, foreground |
| `mapwidth`, `mapheight` | `int` | global | Dimensions of current loaded map (typically 128×128) |
| `TEAM[MAXPLAYERS]` | `teamtype[]` | global | Multiplayer team definitions for current level |
| `walls[MAXWALLTILES]` | `wall_t[]` | global | Wall tile instances for the current level |
| `Clocks[MAXCLOCKS]` | `str_clock[]` | global | Timed switch/door definitions |
| `SPAWNLOC[MAXSPAWNLOCATIONS]` | `_2dvec[]` | global | Player spawn point locations on map |
| `LevelName[80]` | `char[]` | global | Human-readable level name from map header |
| `insetupgame` | `boolean` | global | Flag: true while actively initializing a level |
| `SNAKELEVEL` | `int` | global | Boss level indicator (0=normal, 1-3=snake levels) |
| `cachelist` | `cachetype*` | static | Precache entry array allocated during setup |
| `cacheindex` | `word` | static | Current precache entry count |
| `CachingStarted` | `boolean` | static | Precache system active flag |

## Key Functions / Methods

### SetupGameLevel
- **Signature:** `void SetupGameLevel(void)`
- **Purpose:** Primary level initialization routine called on level start; orchestrates all setup systems
- **Inputs:** None (reads from global `gamestate`)
- **Outputs/Return:** None (initializes global level state)
- **Side effects:** 
  - Loads map via `LoadROTTMap()`
  - Initializes actor/static/door/wall/clock/light lists
  - Precaches graphics and sounds
  - Sets up rendering and physics state
  - Resets game counters (kill count, treasure count, etc.)
- **Calls:** `LoadROTTMap`, `DoLowMemoryConversion`, `DoSharewareConversion`, `SetupPreCache`, `InitializePlayerstates`, `SetupWalls`, `SetupClocks`, `SetupAnimatedWalls`, `SetupDoors`, `SetupPlayers`, `SetupActors`, `SetupElevators`, `PreCache`, `SetupPlayScreen`, `ConnectAreas`, and many others
- **Notes:** Core entry point; handles both new games and loaded games differently; respects difficulty settings

### ReadROTTMap
- **Signature:** `void ReadROTTMap(char *filename, int mapnum)`
- **Purpose:** Load a ROTT binary map file with version checking and RLEZ decompression
- **Inputs:** `filename` (path to .rot or similar file), `mapnum` (0-99 map index in file)
- **Outputs/Return:** None (populates global `mapplanes[]` and `mapwidth/height`)
- **Side effects:** 
  - Allocates zone memory for three map planes
  - Validates RTL signature and version
  - Performs RLEZ decompression of compressed plane data
- **Calls:** `CheckRTLVersion`, `SafeOpenRead`, `SafeRead`, `lseek`, `CA_RLEWexpand`, `Z_Malloc`, `SafeFree`, `close`
- **Notes:** Enforces shareware version restrictions; handles registered/base game tag differences

### LoadROTTMap
- **Signature:** `void LoadROTTMap(int mapnum)`
- **Purpose:** Wrapper that selects and loads appropriate map source (Ted format, alternate directory, or built-in)
- **Inputs:** `mapnum` (map index)
- **Outputs/Return:** None (delegates to specific loaders)
- **Side effects:** Changes working directory temporarily if loading alternate map
- **Calls:** `LoadTedMap`, `LoadAlternateMap`, `ReadROTTMap`, `UL_ChangeDirectory`
- **Notes:** Routes based on `tedlevel` flag and multiplayer/battle mode

### PreCache
- **Signature:** `void PreCache(void)`
- **Purpose:** Execute precache queue and load all graphics/sounds into memory with progress UI
- **Inputs:** None (reads global `cachelist`, `cacheindex`)
- **Outputs/Return:** None (caches data into level heap)
- **Side effects:** 
  - Calls `W_CacheLumpNum` for each precache entry
  - Displays progress bar with LED graphics
  - Plays "dopefish" sound if flag enabled
  - Outputs heap usage statistics in development mode
- **Calls:** `MiscPreCache`, `SortPreCache`, `W_CacheLumpNum`, `Z_HeapSize`, `Z_UsedHeap`, `DrawNormalSprite`, `ShutdownPreCache`, `ConnectAreas`
- **Notes:** Only runs if `CachingStarted==true`; handles both fresh load and loaded game paths

### PreCacheActor
- **Signature:** `void PreCacheActor(int actor, int which)`
- **Purpose:** Precache graphics and sounds for a specific actor type (enemy, boss, etc.)
- **Inputs:** `actor` (actor class ID), `which` (variant/direction for multi-variant actors)
- **Outputs/Return:** None (queues precache entries)
- **Side effects:** Calls `PreCacheLump`, `SD_PreCacheSound`, `SD_PreCacheSoundGroup` for actor-specific assets
- **Calls:** Large switch on actor type; calls `PreCacheGroup`, `W_GetNumForName`, sound precache functions
- **Notes:** Covers ~30+ actor types with variant handling; alternates and difficulty modes supported

### SetupPlayers
- **Signature:** `void SetupPlayers(void)`
- **Purpose:** Parse map icon plane for player spawn tiles and initialize spawn locations
- **Inputs:** None (reads `mapplanes[1]` icon layer)
- **Outputs/Return:** None (populates `SPAWNLOC[]`, spawns player actors)
- **Side effects:** 
  - Spawns player actor via `SpawnPlayerobj()`
  - For multiplayer/battle: calls `SetupTeams()` or random spawn logic
  - Special case: Tag battle mode sets designated "it" player
- **Calls:** `SpawnPlayerobj`, `SetupTeams`, `RespawnPlayerobj`, `PreCachePlayers`, `GameRandomNumber`
- **Notes:** Handles single-player, team-based, and free-for-all modes; respects battle mode flags

### SetupWalls
- **Signature:** `void SetupWalls(void)`
- **Purpose:** Parse background map plane for wall tile definitions and populate wall actor list
- **Inputs:** None (reads `mapplanes[0]`)
- **Outputs/Return:** None (initializes `walls[]` array, updates `tilemap[]`, `actorat[]`)
- **Side effects:** 
  - Sets up wall collision at each wall tile
  - Marks wall-linked sprites in `actorat[]`
  - Precaches wall graphics
  - Handles elevators, special tiles (animated, masked)
- **Calls:** `GetLumpForTile`, `PreCacheLump`, `InitWall` macro
- **Notes:** Critical for collision detection; wall index is used throughout engine

### SetupDoors
- **Signature:** `void SetupDoors(void)`
- **Purpose:** Parse background plane for door tiles and spawn door objects
- **Inputs:** None (reads `mapplanes[0]`)
- **Outputs/Return:** None (spawns door actors via `SpawnDoor()`)
- **Side effects:** 
  - Initializes door actor with lock state
  - Updates `tilemap[]` with door references
  - Respects map-plane-2 for door lock flags
- **Calls:** `SpawnDoor`, various tile range checks
- **Notes:** Handles multiple door types; locked state determined by MAPSPOT(i,j,2)

### SetupActors
- **Signature:** `void SetupActors(void)`
- **Purpose:** Parse icon plane for enemy actor spawn tiles and instantiate them
- **Inputs:** None (reads `mapplanes[1]` icon layer)
- **Outputs/Return:** None (spawns actor objects)
- **Side effects:** 
  - Respects difficulty setting for hard-only enemies
  - Spawns stand/patrol/sneaky guard variants
  - Spawns bosses and special enemy types
  - Precaches actor assets
- **Calls:** Large switch statement calling `SpawnStand`, `SpawnPatrol`, `SpawnSneaky`, `PreCacheActor`, `SpawnMultiSpriteActor`, `SpawnSnake`
- **Notes:** Comprehensive 50+ actor tile types; handles alternate/hard variations

### SetupMaskedWalls
- **Signature:** `void SetupMaskedWalls(void)`
- **Purpose:** Spawn masked/transparent wall objects with passability flags
- **Inputs:** None (reads `mapplanes[0]`)
- **Outputs/Return:** None (spawns masked wall objects)
- **Side effects:** 
  - Calls `SpawnMaskedWall()` with passability/blocking mode flags
  - Handles platforms with multiple height levels
  - Supports shootable/non-shootable variants
- **Calls:** `SpawnMaskedWall` with MW_* flag combinations
- **Notes:** Wide variety of wall types (glass, wood, platforms, railings); passability flags control interaction

### SetupClocks
- **Signature:** `void SetupClocks(void)`
- **Purpose:** Parse icon plane for timed switch/door definitions
- **Inputs:** None (reads `mapplanes[1]` and `mapplanes[2]` special fields)
- **Outputs/Return:** None (populates `Clocks[]` array)
- **Side effects:** 
  - Decodes BCD time values from map-plane-2
  - Calculates activation/deactivation duration in vblank ticks
  - Links clocks to switch tiles
- **Calls:** `FindTimeTile`, clock linking functions
- **Notes:** Times stored as BCD; converted to engine ticks (VBLCOUNTER)

### GetWallIndex
- **Signature:** `int GetWallIndex(int texture)`
- **Purpose:** Translate raw wall texture ID to wall index for rendering/collision
- **Inputs:** `texture` (raw texture value from MAPSPOT)
- **Outputs/Return:** Wall tile index (0-104+)
- **Side effects:** None
- **Calls:** `W_GetNumForName` (for wall/exit/elevator start anchors)
- **Notes:** Decodes texture bit patterns; handles special textures (0x1000 bit, elevator, exit gates)

## Control Flow Notes
**Level Load Sequence:**
1. `SetupGameLevel()` called on level change
2. Load map file → `LoadROTTMap()` → `ReadROTTMap()` + RLEZ decompression
3. Apply version conversions (shareware/registered/low-memory)
4. Initialize subsystems in order:
   - Actor/static/door lists cleared (if not loaded game)
   - `SetupWalls()` → wall collision grid
   - `SetupClocks()` → timed switches
   - `SetupAnimatedWalls()` → animated tile states
   - `SetupDoors()` → door objects
   - `SetupPlayers()` → player spawns
   - `SetupActors()` → enemies
   - `SetupElevators()` → elevator logic
   - `SetupPushWalls()` + `SetupPushWallLinks()` → pushwall mechanics
   - `SetupMaskedWalls()` → transparent geometry
   - `SetupInanimateActors()` → hazards/environmental objects
   - `SetupLights()` → dynamic lighting
5. `PreCache()` → load graphics/sounds asynchronously with UI
6. `SetupScreen()` → finalize rendering state

## External Dependencies
- **Map I/O & Loading:**
  - `w_wad.h`: `W_CacheLumpNum`, `W_GetNumForName`, `W_LumpLength`
  - `z_zone.h`: `Z_Malloc`, `Z_Heap*`, zone memory manager
  - Standard C I/O: `<stdio.h>`, `<io.h>` (DOS `lseek`, `read`, `close`)
- **Actor/Object Spawning:**
  - `rt_actor.h`, `rt_stat.h`, `rt_door.h`: Spawn functions for all entity types
  - `rt_playr.h`: Player state initialization
- **Audio/Music:**
  - `rt_sound.h`: `SD_PreCacheSound*`, `MU_StartSong` (music system)
- **Graphics/Rendering:**
  - `rt_vid.h`, `rt_draw.h`, `rt_scale.h`: Display functions (`DrawNormalSprite`, `VW_UpdateScreen`)
  - `modexlib.h`, `engine.h`: Low-level graphics primitives
- **Game Systems:**
  - `rt_def.h`: Core constants, macros (MAPSPOT, AREATILE, etc.)
  - `rt_util.h`, `rt_cfg.h`: Utility/config functions
  - `rt_floor.h`, `rt_view.h`, `rt_main.h`: Other game systems
- **Utilities:**
  - `watcom.h`: Fixed-point math macros (`FixedMulShift`)
  - `develop.h`, `rt_debug.h`: Debug/development features
