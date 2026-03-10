# rott/rt_stat.c

## File Purpose
Manages static objects (items, decorations, environmental hazards, lights) in the game world. Handles spawning, removal, animation, lighting effects, and persistence/respawn mechanics for 91 distinct static object types.

## Core Responsibilities
- Spawn and remove static objects at tile coordinates with height/type parameters
- Maintain active/inactive and free object pools via doubly-linked lists
- Animate sprite frames and wall textures via per-frame updates
- Manage light sources and compute light influence on surrounding tiles
- Track respawning items with countdown timers and deferred creation
- Serialize/deserialize static objects and switches to game save format
- Cache sprite frames and sounds for pre-loaded static types
- Handle switch state transitions and touch-plate interactions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `statobj_t` | struct | Static object instance (position, sprite, flags, animation state, list ptrs) |
| `respawn_t` | struct | Deferred respawn entry with countdown and original state |
| `statinfo` | struct | Metadata for a stat type (sprite, flags, tictime, hitpoints, ammo) |
| `animwall_t` | struct | Active wall animation (ticcount, frame index, texture) |
| `stat_t` | enum | 91 distinct static object type IDs (ylight, pedkey, weapon, hazard, etc.) |
| `awallinfo_t` | struct (static) | Wall animation template (tictime, frame count, lump name) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FIRSTSTAT`, `LASTSTAT` | `statobj_t*` | global | Head/tail of master doubly-linked list of all statics |
| `firstactivestat`, `lastactivestat` | `statobj_t*` | global | Active animation queue (sprites requiring per-frame updates) |
| `firstemptystat`, `lastemptystat` | `statobj_t*` | global | Pool of freed static objects awaiting reuse |
| `sprites[MAPSIZE][MAPSIZE]` | `statobj_t*` array | global | 2D spatial index: tile coordinates → sprite pointer |
| `switches[MAXSWITCHES]` | `wall_t` array | global | Switch state array (64 max) |
| `lastswitch` | `wall_t*` | global | Pointer to next free switch slot |
| `firstrespawn`, `lastrespawn` | `respawn_t*` | global | Respawn queue (countdown timers for item reappearance) |
| `animwalls[MAXANIMWALLS]` | `animwall_t` array | global | Active wall animation instances (17 max) |
| `animwallsinfo[MAXANIMWALLS]` | `awallinfo_t` array | static | Wall animation metadata (tictime, frame count, lump name) |
| `statcount` | int | global | Total count of spawned statics |
| `stats[NUMSTATS]` | `statinfo` array | global | Lookup table for 91 stat types (metadata) |
| `opposite[9]` | `dirtype` array | global | Direction reversal lookup (east→west, etc.) |
| `diagonal[9][9]` | `dirtype` array | global | Diagonal direction computation table |

## Key Functions / Methods

### SpawnStatic
- Signature: `void SpawnStatic(int tilex, int tiley, int mtype, int zoffset)`
- Purpose: Create a static object instance at a tile location during level load or respawn.
- Inputs: tile X/Y coordinates, stat type index (mtype), Z height offset (byte encoded or -1 for default)
- Outputs/Return: Allocates and links `statobj_t` into lists; no return value.
- Side effects: Allocates memory (Z_LevelMalloc or reuses free pool), modifies `FIRSTSTAT`/`LASTSTAT`, updates sprite spatial index, increments weapon counters, triggers sound/sprite precaching.
- Calls: `Z_LevelMalloc`, `RemoveFromFreeStaticList`, `AddStatic`, `MakeStatActive` (for mines), `Set_NewZ_to_MapValue`, `PreCacheStaticFrames`, `PreCacheStaticSounds`, `FindEmptyTile` (on area error).
- Notes: Handles battle-mode filtering (spawn opts, mines vs health), standard pole height hack (zoffset 14–17→−1), platform/bridge height lookup, shareware sprite substitution. Marks newly-spawned inert objects non-blocking unless otherwise flagged.

### RemoveStatic
- Signature: `void RemoveStatic(statobj_t *stat)`
- Purpose: Deactivate and unlink a static object; optionally queue for respawn.
- Inputs: Pointer to static object to remove.
- Outputs/Return: None.
- Side effects: Unlinks from active/master lists, decrements weapon count, adds to free pool, may create respawn entry (if FL_RESPAWN set and battle respawn enabled), triggers sound.
- Calls: `MakeStatInactive`, `Z_LevelMalloc` (respawn), `AddRespawnStatic`, `AddToFreeStaticList`, decrements `statcount`.
- Notes: Respawn deferred if item is weapon and weapon persistence disabled. Respawn entry stores original Z coordinate and linked_to index.

### CheckCriticalStatics
- Signature: `void CheckCriticalStatics(void)`
- Purpose: Per-frame update for respawn timers and solid-color (damage) object transitions.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Decrements respawn countdown; on expiry, calls `SpawnStatic` and `SpawnNewObj` (for spawn effect particle), triggers respawn sound. For solid-color statics, increments hitpoints; on overflow, removes.
- Calls: `RemoveStatic` (on countdown expiry or hitpoint overflow), `SpawnStatic`, `SpawnNewObj`, `MakeStatActive`, `SD_PlaySoundRTP`, `RemoveRespawnStatic`.
- Notes: Solid-color transitions require precise hitpoint thresholds (SOLIDCOLORINCREMENT, MAXFIRECOLOR).

### DoSprites
- Signature: `void DoSprites(void)`
- Purpose: Per-frame animation update for all active sprites (advance ticcount, update frame).
- Inputs: None.
- Outputs/Return: None.
- Side effects: Modifies ticcount and shapenum of each active static; may remove one-time animations (missmoke, rubble, woodfrag); triggers FL_BACKWARDS flag logic.
- Calls: `RemoveStatic` (for terminal animations like missmoke), implicit frame computation.
- Notes: Special cases: missmoke removed at frame end, rubble/woodfrag deactivate at specific frames, backwards flag reverses frame count.

### TurnOnLight / TurnOffLight
- Signature: `void TurnOnLight(int i, int j)` / `void TurnOffLight(int tilex, int tiley)`
- Purpose: Update lightmap for a tile based on surrounding wall geometry.
- Inputs: Tile coordinates.
- Outputs/Return: None.
- Side effects: Calls `SetLight()` (graphics) for center and adjacent tiles with color values based on wall pattern (cardinal/diagonal).
- Calls: `DoLights` (from TurnOffLight), `SetLight` (graphics backend), `IsLight` (local check).
- Notes: TurnOnLight has 8 corner cases (horizontal, vertical, L-shapes) plus default grid; TurnOffLight delegates via DoLights which checks all 8 neighbors.

### ActivateLight / DeactivateLight
- Signature: `void ActivateLight(int light)` / `void DeactivateLight(int light)`
- Purpose: Toggle light sprite state and update lightmap.
- Inputs: Pointer (cast from `statobj_t*`).
- Outputs/Return: None.
- Side effects: Increments/decrements shapenum, sets/clears FL_LIGHTON flag, calls TurnOnLight/TurnOffLight.
- Calls: `TurnOnLight`, `TurnOffLight`.

### SaveStatics / LoadStatics
- Signature: `void SaveStatics(byte **buffer, int *size)` / `void LoadStatics(byte *buffer, int size)`
- Purpose: Serialize static objects to save buffer / deserialize from load buffer.
- Inputs: Buffer pointer, size (or size for load).
- Outputs/Return: Populates buffer and size.
- Side effects: Allocates buffer (SafeMalloc), copies stat fields to `saved_stat_type` format, indexes respawn links.
- Calls: `SafeMalloc`, `Z_LevelMalloc`, `InitStaticList`, `AddStatic`, `PreCacheStaticFrames`, `PreCacheStaticSounds`, `SetupBulletHoleLink` (for bullet holes).
- Notes: Respawn links and bullet-hole special cases handled; sprite spatial index rebuilt on load.

### AnimateWalls
- Signature: `void AnimateWalls(void)`
- Purpose: Per-frame update for all active wall animations.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Decrements ticcount; on ≤0, advances frame counter and resets ticcount; wraps to basetexture after last frame.
- Calls: None (self-contained).
- Notes: Skipped if DoPanicMapping() true; ticcount may go negative multiple frames before wrap.

### PreCacheStaticFrames / PreCacheStaticSounds
- Signature: `void PreCacheStaticFrames(statobj_t *temp)` / `void PreCacheStaticSoundS(int itemnumber)`
- Purpose: Pre-load sprite graphics and sound effects for a stat type.
- Inputs: Stat object or item number.
- Outputs/Return: None.
- Side effects: Calls PreCacheLump/PreCacheGroup (graphics), SD_PreCacheSound (audio).
- Calls: `W_GetNumForName`, `PreCacheLump`, `PreCacheGroup`, `SD_PreCacheSoundGroup`, `SD_PreCacheSound`.
- Notes: PreCacheStaticFrames conditionally loads female/black player sprites based on locplayerstate; handles special case gfx for pedestal keys, bat, knife statue, weapons, godmode. PreCacheStaticSounds uses large switch for item-specific sounds.

### List Management (AddStatic, AddToFreeStaticList, etc.)
- Signature: `void AddStatic(statobj_t *stat)` / `void AddToFreeStaticList(statobj_t *stat)` / `void MakeStatActive/Inactive(statobj_t *x)`
- Purpose: Insert/remove from master, free, or active doubly-linked lists.
- Inputs: Pointer to static object.
- Outputs/Return: None.
- Side effects: Updates list pointers (statnext/statprev for master, nextactive/prevactive for active).
- Calls: None (pointer manipulation only).
- Notes: All lists are doubly-linked; NULL checks on first/last transitions.

## Control Flow Notes

**Initialization Phase:**
- `InitStaticList()` zeroes all list heads/tails, clears sprite map, initializes bullet holes array.
- `InitAnimatedWallList()` disables all wall animations.

**Spawn Phase (during level load):**
- Ted map parser calls `SpawnStatic()` for each static entity.
- `SetupAnimatedWall()` activates wall animations referenced by map.
- `SaveStatics()`/`LoadStatics()` reconstruct statics from save file.

**Per-Frame Update:**
1. `DoSprites()` advances animation frame and ticcount for active statics.
2. `AnimateWalls()` advances wall animation frame.
3. `CheckCriticalStatics()` processes respawn timers and solid-color transitions.

**Removal/Respawn:**
- `RemoveStatic()` delinks and optionally queues for respawn.
- `CheckCriticalStatics()` spawns respawned items after countdown.

**Lighting (interactive):**
- Actor/player interaction triggers `ActivateLight()`/`DeactivateLight()`.
- Light switch changes call `TurnOnLight()`/`TurnOffLight()` to recompute lightmap.

**Save/Load:**
- `SaveStatics()` called before level exit.
- `LoadStatics()` called after level load (rebuilds indices).

## External Dependencies
- **Memory**: `z_zone.h` — `Z_LevelMalloc()`, `Z_Free()`
- **Graphics**: `lumpy.h` — sprite/patch structures; `rt_draw.h` — `PreCacheLump()`, `PreCacheGroup()`, `SetLight()` 
- **Audio**: `rt_sound.h` — `SD_PreCacheSound()`, `SD_PlaySoundRTP()`
- **Map/Level**: `rt_ted.h` — wall/tile structures; global `tilemap`, `mapplanes`, `MAPSPOT()`, `actorat`, `spotvis`, `LightsInArea`
- **Entities**: `rt_main.h` — `locplayerstate`, `gamestate`, `MISCVARS`, `new`, `tics`
- **Utilities**: `rt_util.h` — `GameRandomNumber()`, `FindEmptyTile()`, `IsPlatform()`, `PlatformHeight()`
- **Game State**: `rt_net.h`, `rt_menu.h`, `rt_view.h` (battle mode, player info)
- **Actors**: `rt_door.h`, actor spawning (`GetNewActor()`, `NewState()`, `MakeActive()`, `SpawnNewObj()`)
