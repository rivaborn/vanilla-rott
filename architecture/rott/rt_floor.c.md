# rott/rt_floor.c

## File Purpose
Implements floor, ceiling, and parallax sky rendering for the 3D raycaster engine. Handles texture mapping, light shading, and horizontal span rasterization for the ground and overhead planes in the game world.

## Core Responsibilities
- Initialize and manage floor/ceiling/sky texture data at level load
- Render the main floor and ceiling planes via horizontal scanline traversal
- Implement parallax-scrolling sky rendering with configurable horizon height
- Calculate per-pixel light levels based on distance (fog/brightness)
- Manage VGA plane switching for chunked 256-color graphics
- Load and process texture lumps from the WAD resource system

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sky` | int | global | Sky type ID (0=none, 1–6=parallax sky variant) |
| `mr_xstep` | int | global | X stepping for texture mapping rotation |
| `mr_ystep` | int | global | Y stepping for texture mapping rotation |
| `mr_xfrac` | int | global | X fractional position during row draw |
| `mr_yfrac` | int | global | Y fractional position during row draw |
| `mr_dest` | byte* | global | Current VGA destination buffer pointer |
| `mr_src` | byte* | global | Current texture source pointer |
| `floor` | byte* | static | Floor texture data (cached lump) |
| `ceiling` | byte* | static | Ceiling texture data (cached lump) |
| `xstarts` | int[MAXVIEWHEIGHT] | static | Leftmost x for each horizontal scanline |
| `skysegs` | byte*[MAXSKYSEGS] | static | Pointers to rotated sky column segments |
| `skydata` | byte*[MAXSKYDATA] | static | Raw sky texture data buffers |
| `horizonheight` | int | static | Horizon pixel offset from center (1–8) |
| `centerskypost` | int | static | Vertical center of sky column |
| `oldsky` | int | static | Previously loaded sky ID (for cleanup) |

## Key Functions / Methods

### DrawSky
- Signature: `void DrawSky(void)`
- Purpose: Render parallax sky in the ceiling area above wall clips. Handles lightning/fog shading and processes two columns per loop with VGA plane splitting for efficiency.
- Inputs: Global `viewangle`, `posts[].ceilingclip`, `pixelangle[]`, `viewx`, `viewy`, `player->z`
- Outputs/Return: Writes directly to video memory via `bufferofs`
- Side effects: Updates `shadingtable`, modifies VGA plane mask
- Calls: `VGAMAPMASK()`, `VGAWRITEMAP()`, `DrawSkyPost()` (asm)
- Notes: Skips columns with ceiling clip ≤ 0; scales sky offset based on player height and horizon; splits rendering into two planes when `doublestep > 0` for double-wide pixels

### DrawFullSky
- Signature: `void DrawFullSky(void)`
- Purpose: Fill entire viewscreen with parallax sky (used when no walls are present).
- Inputs: Global `viewangle`, `pixelangle[]`, `viewx`, `viewy`, `player->z`
- Outputs/Return: Writes to full viewport in video memory
- Side effects: Sets `shadingtable` for lightning/fog; shifts `bufferofs` by `screenofs`
- Calls: `VGAWRITEMAP()`, `DrawSkyPost()` (asm)
- Notes: Covers all `viewwidth` columns at full `viewheight`; used for exterior sky-only views

### MakeSkyTile
- Signature: `void MakeSkyTile(byte *tile)`
- Purpose: Generate a single 64×64 sky tile by sampling from precomputed sky columns with fixed-point stepping.
- Inputs: `tile` pointer to 4KB output buffer
- Outputs/Return: Fills `tile` with scaled sky image data
- Side effects: None
- Calls: Memory write
- Notes: Samples from `skysegs[]` with srcstep=200<<10; simple point-sampling without filtering

### MakeSkyData
- Signature: `void MakeSkyData(void)`
- Purpose: Merge two sky texture lumps (top/bottom) into a single 256×400-byte buffer for fast lookup.
- Inputs: `skydata[0]` and `skydata[1]` (cached lumps)
- Outputs/Return: Allocates new buffer and updates `skydata[0]`
- Side effects: Allocates heap memory; frees old sky data if `oldsky > 0`
- Calls: `SafeMalloc()`, `memcpy()`
- Notes: Interleaves rows: 200 bytes from skydata[1], 200 from skydata[0] for each iteration; original lumps are freed after

### GetFloorCeilingLump
- Signature: `int GetFloorCeilingLump(int num)`
- Purpose: Map numeric floor/ceiling ID (1–16) to corresponding WAD lump index.
- Inputs: `num` (1–16)
- Outputs/Return: Lump index from WAD namespace
- Side effects: None
- Calls: `W_GetNumForName()`
- Notes: Errors on num out of range; lump names are "FLRCL1" through "FLRCL16"

### SkyExists
- Signature: `boolean SkyExists(void)`
- Purpose: Detect whether current map contains a parallax sky by checking map spot 1,0,0.
- Inputs: Map plane 1 at (1,0)
- Outputs/Return: `true` if value ≥ 234, else `false`
- Side effects: None
- Calls: `MAPSPOT()`
- Notes: Sky sprite IDs start at 234 in the map

### SetPlaneViewSize
- Signature: `void SetPlaneViewSize(void)`
- Purpose: Initialize floor, ceiling, and sky rendering for a new level. Loads textures, processes sky parameters (type, horizon height, lightning), and pre-computes sky column lookup tables.
- Inputs: Map data (MAPSPOT), game state
- Outputs/Return: Initializes static file state (`floor`, `ceiling`, `skysegs`, etc.)
- Side effects: Allocates and caches lumps; frees previous sky data; modifies `lightning` flag
- Calls: `MAPSPOT()`, `GetFloorCeilingLump()`, `W_CacheLumpNum()`, `W_GetNumForName()`, `Error()`, `MakeSkyData()`, `SafeFree()`, `SafeMalloc()`
- Notes: Checks map sprite at (1,0,1) for horizon height (90–97 → 1–8, or 450–457 → 9–16); checks (4,0,1) for lightning sprite (377). Sky IDs computed as MAPSPOT(1,0,0) - 233. Non-sky maps use ceiling from plane 1; floor always from plane 0 offset −179

### SetFCLightLevel
- Signature: `void SetFCLightLevel(int height)`
- Purpose: Select the appropriate shading colormap row based on distance and environmental conditions (gas, fog, fullbright).
- Inputs: `height` (distance-dependent depth value)
- Outputs/Return: Updates global `shadingtable`
- Side effects: Modifies `shadingtable` pointer
- Calls: None
- Notes: Gas mode uses `greenmap`; fullbright skips shading; fog mode scales linearly with height

### DrawHLine
- Signature: `void DrawHLine(int xleft, int xright, int yp)`
- Purpose: Render a single horizontal span of floor or ceiling by computing texture coordinates (u,v stepping) and dispatching per-plane row draws.
- Inputs: `xleft`, `xright` (screen x range), `yp` (screen y); global `posts[]`, `viewx`, `viewy`, `player->z`, `viewsin`, `viewcos`
- Outputs/Return: Writes to video memory
- Side effects: Sets `shadingtable`, `mr_xstep`, `mr_ystep`, `mr_xfrac`, `mr_yfrac`, `mr_dest`, `mr_count`; modifies VGA plane masks
- Calls: `SetFCLightLevel()`, `FixedMulShift()`, `VGAMAPMASK()`, `VGAWRITEMAP()`, `DrawRow()` (asm)
- Notes: Distinguishes floor (y > centery) and ceiling (y < centery) by distance formula; early-exit if yp == centery; handles 4-plane VGA chunking with optional double-stepping; stores column x in `xstarts[]` during traversal

### DrawPlanes
- Signature: `void DrawPlanes(void)`
- Purpose: Main entry point for floor/ceiling rendering. Builds a scanline-span list and rasterizes all visible floor and ceiling regions.
- Inputs: Global `viewwidth`, `viewheight`, `posts[]`, `sky`
- Outputs/Return: Renders floor/ceiling to video memory
- Side effects: Modifies `xstarts[]` array and calls DrawSky/DrawHLine which modify video state
- Calls: `DrawSky()`, `DrawHLine()`
- Notes: If `sky` is nonzero, renders parallax sky; otherwise, for ceiling: top-to-bottom scanline tracking with span collection; for floor: bottom-to-top. Uses edge-stepping to detect and draw spans between wall edges and screen boundaries

## Control Flow Notes
**Level init:** `SetPlaneViewSize()` loads floor/ceiling/sky at level load time.

**Frame render:** `DrawPlanes()` called each frame after walls are rasterized. Invokes `DrawSky()` if sky ≠ 0; otherwise draws floor and ceiling via `DrawHLine()` spans.

**Per-span:** `DrawHLine()` computes perspective-correct texture mapping (u,v steps based on distance) and dispatches assembly `DrawRow()` calls, one per VGA plane.

## External Dependencies
- **WAD Loading:** `W_CacheLumpNum()`, `W_GetNumForName()` (w_wad.h)
- **Memory:** `SafeMalloc()`, `SafeFree()` (z_zone.h)
- **Engine globals:** `viewangle`, `viewx`, `viewy`, `viewsin`, `viewcos`, `player`, `posts[]`, `pixelangle[]`, `bufferofs`, `ylookup[]`, `MAPSPOT()` (defined elsewhere)
- **Rendering:** `shadingtable`, `colormap`, `greenmap`, `basemaxshade`, `lightninglevel`, `fog`, `fulllight` (rt_view.h, rt_draw.h)
- **VGA I/O:** `VGAWRITEMAP()`, `VGAMAPMASK()` (modexlib.h)
- **Assembly functions:** `DrawSkyPost()`, `DrawRow()` (rt_fc_a.h, rt_fc_a.asm)
- **Fixed-point math:** `FixedMulShift()` (watcom.h)
- **Constants:** `MAXVIEWHEIGHT`, `MAXSKYSEGS`, `MAXSKYDATA`, `MINSKYHEIGHT`, `FINEANGLES` (_rt_floo.h, rt_def.h)
