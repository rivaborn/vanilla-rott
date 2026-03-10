# rott/engine.c

## File Purpose
Implements the core ray-casting renderer that determines which walls are visible from the player's viewpoint. Uses hierarchical screen-space subdivision and fixed-point grid traversal to efficiently cast rays and populate wall geometry data for rendering.

## Core Responsibilities
- **Ray casting**: Cast rays from player viewpoint through screen pixels to find wall intersections
- **Hierarchical culling**: Use a 4-pixel comb filter with binary subdivision to reduce ray count
- **Tile grid traversal**: Fixed-point arithmetic grid walking to find ray-tile intersections
- **Wall property resolution**: Handle special tile types (doors, windows, animated walls, masked objects) and determine texture coordinates
- **Visibility tracking**: Mark visible areas in spotvis and mapseen arrays for map rendering
- **Texture interpolation**: Linearly interpolate texture and height between nearby cast rays to fill screen gaps

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `wallcast_t` | struct (defined in engine.h) | Stores computed wall post data (texture, height, type, offset) for a screen column |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `posts[321]` | `wallcast_t[]` | global | Screen column data for all 320+ pixels; populated by cast functions |
| `lasttilex`, `lasttiley` | `int` | global | Tile coordinates of last wall intersection (for debugging or state query) |
| `xtilestep`, `ytilestep` | `int` | static | Grid step direction/magnitude (±0x80 or ±1) based on ray direction |
| `c_vx`, `c_vy` | `int` | static | Camera-relative velocity components for current ray in fixed-point |

## Key Functions / Methods

### Refresh
- **Signature**: `void Refresh(void)`
- **Purpose**: Main per-frame rendering pipeline; casts rays at varying densities and fills the posts array.
- **Inputs**: None; uses global camera state (viewx, viewy, viewwidth, viewsin, viewcos).
- **Outputs/Return**: None; modifies `posts[]` array.
- **Side effects**: Calls InitialCast() and Cast() for 0–320 pixel range; updates spotvis/mapseen; modifies posts array.
- **Calls**: `InitialCast()`, `Cast()`, `Interpolate()`, macro `NOTSAMETILE()`.
- **Notes**: Hierarchical approach: coarse 4-pixel casts, then subdivide if adjacent posts differ; full interpolation otherwise. Loop terminates at `viewwidth-4`.

### InitialCast
- **Signature**: `void InitialCast(void)`
- **Purpose**: Cast rays at 4-pixel intervals to establish initial wall geometry coverage.
- **Inputs**: None; uses global view state.
- **Outputs/Return**: None; populates `posts[0..viewwidth]` at 4-pixel intervals; marks spotvis/mapseen.
- **Side effects**: Modifies global `posts[]`, `spotvis[][]`, `mapseen[][]`; sets `xtilestep`, `ytilestep`, `c_vx`, `c_vy`.
- **Calls**: `FixedMul()`, `FixedScale()`, `HitWall()`, `MakeWideDoorVisible()` (extern).
- **Notes**: Grid-walking loop with signed increments. Stops at first non-door solid tile. Handles multi-tile doors via `DF_MULTI` flag. Four-iteration loop over screen with 4-pixel stride.

### Cast
- **Signature**: `void Cast(int curx)`
- **Purpose**: Cast a single ray at arbitrary screen pixel, using same grid-walking logic as InitialCast.
- **Inputs**: `curx` — screen pixel x-coordinate.
- **Outputs/Return**: None; populates `posts[curx]`.
- **Side effects**: Modifies `posts[curx]`, `spotvis[][]`, `mapseen[][]`; sets `xtilestep`, `ytilestep`, `c_vx`, `c_vy`.
- **Calls**: `FixedMul()`, `FixedScale()`, `HitWall()`, `MakeWideDoorVisible()`.
- **Notes**: Nearly identical to InitialCast but for a single ray; ray direction computed from `c_startx`, `c_starty`, `viewsin`, `viewcos`, and pixel offset.

### HitWall
- **Signature**: `void HitWall(int curx, int vertical, int xtile, int ytile)`
- **Purpose**: Resolve wall properties (texture, type, height) for a tile hit; handle special tile types (doors, windows, masked objects, animated walls).
- **Inputs**: `curx` — screen column index; `vertical` — ray direction flag (>0 = horizontal hit, <0 = vertical); `xtile`, `ytile` — grid coordinates.
- **Outputs/Return**: None; populates `posts[curx]` completely.
- **Side effects**: Modifies `posts[curx]`; calls `CalcHeight()` (extern).
- **Calls**: `CalcHeight()`, `IsWindow()` macro, `FixedScale()`.
- **Notes**: Dual-path logic for vertical vs. horizontal hits. Checks tile flags (0x8000=door/masked, 0x4000=masked, 0x2000=window, 0x1000=animated, 0x800=semi-transparent). Clamps texture to [0, 65535]. Applies XOR flip for wall orientation.

### Interpolate
- **Signature**: `void Interpolate(int x1, int x2)`
- **Purpose**: Fill wall post array between two known rays using linear interpolation of texture and height.
- **Inputs**: `x1`, `x2` — screen column indices (typically 4 pixels apart).
- **Outputs/Return**: None; modifies `posts[x1+1..x2-1]`.
- **Side effects**: Overwrites texture and wallheight in intermediate posts; copies posttype, lump, offset, alttile.
- **Calls**: None (direct array access).
- **Notes**: Fixed-point interpolation with 12-bit fractional accumulator for texture, 8-bit for height. Copies non-interpolated fields from x1. Iterates from x1+1 to x2-1.

## Control Flow Notes
This file is part of the **frame/render** phase. `Refresh()` is the entry point, typically called once per game frame to populate the `posts[]` array with wall geometry. The hierarchical approach (comb filter + subdivision) reduces ray casting overhead by exploiting spatial coherence—adjacent screen pixels often see the same wall. Grid traversal and door/window handling make the renderer aware of map structure and special entities.

## External Dependencies
- **Includes/Imports**: rt_def.h (constants), watcom.h (fixed-point math), engine.h (wallcast_t typedef), _engine.h (macros), rt_eng.h, rt_draw.h, rt_door.h, rt_stat.h, rt_ted.h, rt_view.h (declarations assumed).
- **Defined elsewhere**:
  - `tilemap[][]` — tile grid data
  - `doorobjlist[]`, `maskobjlist[]` — entity arrays
  - `animwalls[]` — animated wall texture table
  - `spotvis[][]`, `mapseen[][]` — visibility tracking
  - `viewx`, `viewy`, `viewwidth`, `viewsin`, `viewcos` — camera state
  - `c_startx`, `c_starty` — initial ray direction
  - `CalcHeight()` — compute wall height
  - `MakeWideDoorVisible()` — multi-tile door visibility
  - `FixedMul()`, `FixedScale()` — fixed-point arithmetic (watcom.h)
  - `MAPSPOT()` macro — map plane access
