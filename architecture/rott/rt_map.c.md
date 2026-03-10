# rott/rt_map.c

## File Purpose
Implements interactive map/minimap display and exploration system for Rise of the Triad. Provides zoom-able map views showing level layout, explored areas, actors, sprites, and player position with directional indicator.

## Core Responsibilities
- Render minimap at configurable zoom levels with proper tile scaling
- Draw full-screen map at maximum zoom showing entire level
- Track and visualize explored/seen areas of the map
- Render all map element types (walls, doors, animated walls, masked objects, actors, sprites)
- Handle interactive map mode with keyboard navigation and zoom controls
- Manage map color schemes and display options
- Scale and position sprites appropriately on minimap

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Ptype` | struct | 2D coordinate pair for arrow direction pattern vertices |
| `arrows` | static array | 8 directional arrow patterns (7 vertices each) for player heading indicator |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `tilesize` | int | static | Current pixel size of a map tile on display |
| `xscale`, `yscale` | fixed | static | Scaling factors for viewport dimensions |
| `mapscale` | int | static | Current zoom level (0–5, where 5 is full map) |
| `oldw`, `oldh` | int | static | Saved view dimensions before map mode |
| `skytile` | byte* | static | Cached sky texture data (64×64) if level has sky |
| `mapcolor` | int | static | Current background color index (0–15) |
| `arrows` | Ptype[8][7] | static | Arrow vertex patterns for 8 directions |

## Key Functions / Methods

### DrawMap
- Signature: `void DrawMap(int cx, int cy)`
- Purpose: Render minimap at current zoom level, displaying visible walls, doors, sprites, actors, and player
- Inputs: `cx`, `cy` = camera center position in world coordinates (fixed 16.16)
- Outputs/Return: None; renders to video buffer
- Side effects: Modifies `bufferofs` video memory; calls VGA write modes
- Calls: `VL_ClearBuffer`, `DrawMap_Player`, `DrawMap_Wall`, `DrawMap_AnimatedWall`, `DrawMap_SkyTile`, `DrawMap_Door`, `DrawMap_MaskedWall`, `DrawMap_PushWall`, `DrawMap_Actor`, `DrawMap_Sprite`, `DrawMap_MaskedShape`, `FlipPage`
- Notes: Clips rendering to map bounds [0,127]×[0,127]; skips tiles not in `mapseen`; uses VGA plane writes for 4-bit color

### DrawFullMap
- Signature: `void DrawFullMap(void)`
- Purpose: Render entire level map at maximum zoom (FULLMAP_SCALE=5), showing 1 pixel per tile
- Inputs: None
- Outputs/Return: None; renders to video buffer
- Side effects: Modifies video memory; sets VGA write map planes
- Calls: `SetupFullMap`, `FlipPage`
- Notes: Iterates all 128×128 tiles; uses color lookup table `egacolor[MAP_*COLOR]` for each element type; handles actor/sprite FL_SEEN flag

### DoMap
- Signature: `void DoMap(int cx, int cy)`
- Purpose: Main interactive map mode—allows player to navigate and zoom minimap with keyboard
- Inputs: `cx`, `cy` = player starting position
- Outputs/Return: None
- Side effects: Modifies global view dimensions; allocates/frees `skytile` buffer; swaps video modes; updates keyboard/input state
- Calls: `ShutdownClientControls`, `SetupMapper`, `ChangeMapScale`, `DrawMap`, `DrawMapInfo`, `IN_UpdateKeyboard`, `ReadAnyControl`, `CalcTics`, `DoSprites`, `AnimateWalls`, `ShutdownMapper`, `StartupClientControls`
- Notes: Main loop exits on Tab/Escape or SpaceBall button; handles PgUp/PgDn/+/– for zoom; arrow keys pan at constant rate; Caps+C/X save screenshots; Caps+M reveals all map (dev only); Home recenters view; clamps viewport within bounds

### ChangeMapScale
- Signature: `void ChangeMapScale(int *newx, int *newy, int newmapscale)`
- Outputs/Return: Updates `*newx`, `*newy` to reflect viewport shift; calls `SetupMapScale`
- Purpose: Change zoom level, recenter camera to maintain viewport center position
- Calls: `DrawFullMap`, `SetupMapScale`
- Notes: Bounds-checks `newmapscale` [0, FULLMAP_SCALE]; calls `DrawFullMap` if zooming to max

### SetupMapScale
- Signature: `void SetupMapScale(int s)`
- Purpose: Configure tile size and viewport scale based on zoom level
- Inputs: `s` = zoom level
- Side effects: Updates `tilesize`, `xscale`, `yscale`, `hp_srcstep`
- Calls: None directly
- Notes: `tilesize = 64 >> mapscale`; used by all DrawMap_* functions for positioning

### DrawMap_Player
- Signature: `void DrawMap_Player(int x, int y)`
- Purpose: Draw player position and facing direction on minimap
- Inputs: `x`, `y` = tile coordinates in minimap space
- Calls: `DrawMap_PlayerArrow`, `DrawMap_MaskedShape`
- Notes: Arrow direction randomized if FL_SHROOMS flag set; otherwise computed from `player->angle`

### DrawMap_PlayerArrow
- Signature: `void DrawMap_PlayerArrow(int x, int y, int dir)`
- Purpose: Render directional arrow indicator for player heading
- Inputs: `x`, `y` = tile center; `dir` = direction index [0–7]
- Calls: `VL_DrawLine` (7 times for connected arrow segments)
- Notes: Uses `arrows[dir]` vertex array; scaled by `4-mapscale` for zoom; color 244 (white)

### DrawMap_Wall / DrawMap_AnimatedWall / DrawMap_SkyTile
- Signature: `void DrawMap_Wall(int x, int y, int tile)` / `DrawMap_AnimatedWall` / `DrawMap_SkyTile`
- Purpose: Render standard/animated/sky wall tile to minimap
- Inputs: `x`, `y` = tile coords; `tile` = lump/texture index
- Calls: `W_CacheLumpNum`, `VGAWRITEMAP`, `DrawMapPost`
- Notes: Iterates VGA planes; reads tile lump and outputs via `DrawMapPost`; sky tiles use cached `skytile` buffer

### DrawMap_Door / DrawMap_MaskedWall / DrawMap_PushWall
- Purpose: Render specialized tile types with state-dependent textures
- Inputs: `x`, `y` = tile coords; `tile` = object list index or texture bits
- Calls: `DrawMap_Wall`, `DrawMap_MaskedShape`, `IsPlatform`, `IsWindow`
- Notes: Door handles lock state; masked wall checks platform/passable flags; push wall decodes texture bits

### DrawMap_Actor / DrawMap_Sprite
- Signature: `void DrawMap_Actor(int x, int y, objtype *a)` / `DrawMap_Sprite(int x, int y, statobj_t *s)`
- Purpose: Draw enemy actor or static sprite if it has been seen (FL_SEEN)
- Inputs: Entity pointers
- Calls: `DrawMap_MaskedShape`
- Notes: Skip if not FL_SEEN; translucent flag passed to shape renderer

### DrawMap_MaskedShape
- Signature: `void DrawMap_MaskedShape(int x, int y, int lump, int type)`
- Purpose: Draw scaled sprite/shape centered on tile, handling transparency
- Calls: `DrawPositionedScaledSprite`
- Notes: Converts tile coords to pixel center; uses `tilesize` as scale factor

### SetupMapper / ShutdownMapper
- Purpose: Initialize and teardown map mode state
- Setup: Saves view dimensions; allocates sky tile buffer if needed; sets video mode
- Shutdown: Restores video; frees sky tile; restores view dimensions; re-enables mouse if present
- Calls: `FixMapSeen`, `SafeMalloc`, `MakeSkyTile`, `VL_ClearVideo`, `SetupScreen`, `SafeFree`, `PollMouseMove`

### CheatMap / FixMapSeen
- Purpose: Manage explored area tracking
- CheatMap: Reveal entire map (development cheat)
- FixMapSeen: Fill in unexplored tiles adjacent to explored tiles (fog-of-war smoothing)
- Notes: FixMapSeen uses complex logic checking corner/edge neighbors and diagonal passability

## Control Flow Notes
- **Initialization**: `SetupMapper` prepares video and buffers before entering map mode
- **Main Loop**: `DoMap` runs interactive map—polls input, updates viewport position, renders each frame
- **Rendering**: `DrawMap` (minimap) or `DrawFullMap` (full map); then `DrawMapInfo` for UI text
- **Shutdown**: `ShutdownMapper` restores video mode and cleans up allocations
- Zoom is controlled by `SetupMapScale` → updates `tilesize`; viewport clamping prevents off-map scrolling

## External Dependencies
- Notable includes: `rt_def.h` (common types), `rt_draw.h`, `rt_dr_a.h` (low-level drawing), `w_wad.h` (lump caching), `rt_door.h`, `modexlib.h` (VGA), `rt_vid.h`, `rt_in.h` (input), `z_zone.h` (memory)
- Defined elsewhere: `tilemap[][]`, `actorat[][]`, `sprites[][]`, `doorobjlist[]`, `maskobjlist[]`, `animwalls[]`, `mapseen[][]`, `player` (global), `bufferofs`, `ylookup[]`, `egacolor[]`, `sky`, `mapwidth`, `mapheight`, `shapestart`, `Keyboard[]`, `RandomNumber`, `W_CacheLumpNum`, `IsPlatform`, `IsWindow`, `DrawPositionedScaledSprite`, `VL_DrawLine`, `VL_ClearBuffer`, `FlipPage`, `CalcTics`
