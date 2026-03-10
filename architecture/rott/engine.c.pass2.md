# rott/engine.c — Enhanced Analysis

## Architectural Role

This file is the heart of the **view/render subsystem**, implementing a classic DDA-style raycasting pipeline that bridges camera state (position, angle) to wall geometry for drawing. It consumes tilemap grid data and entity metadata (doors, masked objects), and populates `posts[]` which feeds directly into the column-based drawing pipeline in `rt_draw.c`. The hierarchical 4-pixel comb filter + binary subdivision strategy exploits spatial coherence to reduce per-frame ray casts by ~75%, a critical optimization for 1994-era performance budgets.

## Key Cross-References

### Incoming (who depends on this file)
- **Frame loop / main renderer**: `Refresh()` is the entry point; likely called once per frame from `rt_draw.c` or game loop
- **Drawing pipeline**: `rt_draw.c` reads `posts[]` array post-computation to rasterize wall columns (confirmed: `CalcHeight()` in `rt_draw.c` is called by `HitWall()`)
- **Visibility system**: `rt_map.c` reads `spotvis[][]` and `mapseen[][]` for automap rendering

### Outgoing (what this file depends on)
- **Fixed-point math** (watcom.h): `FixedMul()`, `FixedScale()` — core to DDA grid traversal
- **Wall height calculation** (rt_draw.c): `CalcHeight()` — perspective-corrected wall height per column
- **Door/multi-tile handling**: `MakeWideDoorVisible()` (extern, likely rt_door.c) — marks adjacent tiles visible for wide doors
- **Macro utilities**: `NOTSAMETILE()`, `MAPSPOT()`, `IsWindow()` — tile classification
- **Global map state**: `tilemap[][]` (tile IDs), `doorobjlist[]`, `maskobjlist[]` (entity metadata), `animwalls[]` (animated texture LUT)
- **Global camera state**: `viewx`, `viewy`, `viewwidth`, `viewsin`, `viewcos`, `c_startx`, `c_starty` — set by view/control subsystem
- **Global visibility arrays**: `spotvis[][]`, `mapseen[][]` — written to track traced cells

## Design Patterns & Rationale

**Hierarchical Adaptive Casting**: The comb-filter approach in `Refresh()` is a textbook spatial coherence optimization. Cast every 4th pixel initially; if adjacent 4-pixel blocks see different tiles, binary-subdivide (cast at +2, then +1/+3 if needed). This trades computation for a single conditional check per 4-pixel region. Trade-off: occasional texture discontinuities on sharp diagonal walls, but imperceptible at 320×200.

**DDA-style Grid Traversal**: Both `InitialCast()` and `Cast()` use a fixed-point **grid stepping algorithm** similar to DDA (Digital Differential Analyzer):
- Pre-compute direction (`thedir[0/1]` ∈ {-1, +1}) and step magnitude (`xtilestep`/`ytilestep`)
- Track grid position and parametric distance (`cnt`) to next grid crossing
- Step whichever axis is closer, updating `cnt` and checking for tile hits
- This avoids floating-point and yields clean tile-aligned intersections

**Tile Type Bitmask System**: Tiles use packed flags (0x8000=door/masked, 0x4000=masked, 0x2000=window, 0x1000=animated, 0x800=semi-transparent). `HitWall()` checks these in a cascading if-else to resolve the actual displayable texture. This is compact but makes adding new tile types non-trivial.

**Texture Coordinate Flipping**: XOR operations (`texture ^= 0xffff`) flip texture U-coordinates based on ray direction and hit orientation. This handles mirror symmetry in grid-aligned geometry without extra conditionals—a clever fixed-point trick.

## Data Flow Through This File

1. **Input**: Global camera state (position `viewx/y`, angle encoded as `viewsin/cos`, viewport width) set by control/view subsystem
2. **Coarse rays**: `InitialCast()` casts 4-pixel-spaced rays, populating `posts[]` at indices 0, 4, 8, ..., viewwidth
3. **Visibility side effect**: Each ray marks `spotvis[][]` and `mapseen[][]` for that tile and traversed cells
4. **Subdivision**: `Refresh()` checks adjacent posts; if tiles differ, `Cast()` fills gaps at intermediate pixels
5. **Interpolation**: `Interpolate()` linearly fills remaining gaps with texture/height from neighbors
6. **Output**: `posts[]` array fully populated; consumed by drawing pipeline each frame
7. **State persistence**: `lasttilex/y` available for debug/state queries (values written in `HitWall()` but not shown in this file)

## Learning Notes

**Classical Raycasting Era Design**: This is pure early-90s raycasting (Wolfenstein 3D, Catacomb 3-D lineage). No spatial partitioning (BSP/portal), no hierarchical culling beyond the comb filter. Grid is globally searchable; every ray walks the grid from player to first solid hit.

**Fixed-Point Conventions**: The codebase uses 16-bit fixed-point (shift 16 for integer part). Watch the `<<16`, `>>16` operators and macro names like `FixedMul()`. Texture coordinates use 12-bit fractional precision (`<<12` in `Interpolate()`), height uses 8-bit (`<<8`). This is necessary for sub-pixel accuracy without FPU overhead on 486/Pentium era hardware.

**Modern Engine Contrasts**:
- No entity collision/visibility pre-computation (raycasting is on-demand per frame)
- No spatial hashing or quadtree; linear tilemap scan
- No SIMD; scalar DDA stepping
- Tile flags are bit-packed rather than per-tile objects; limited extensibility
- Visibility (`spotvis/mapseen`) is a side effect rather than a dedicated visibility system

**Idiomatic Patterns**:
- `grid[0]`, `grid[1]` represent X and Y; the `index=(cnt>=0)` trick selects which axis to step (avoids branch)
- `thedir[index]` applies the direction; `incr[index]` applies the magnitude
- The `do...while(1)` loops with explicit `break` on hit are typical for grid walkers of this era

## Potential Issues

**Coordinate Precision Edge Cases**: `posts[curx].texture` is clamped to [0, 65535] post-XOR. If a ray hits exactly at a tile boundary, rounding may cause texture seams. The commented-out `&= 0xffff` suggests prior issues with wraparound.

**Tile Type Ambiguity**: The bitmask checks (0x4000, 0x2000, 0x1000, 0x800) are cascading if-else. Order matters; a tile with multiple flags will only match the first condition. Not clear if this is by design or a latent bug. The tile structure in `tilemap[][]` encoding is opaque from this file alone.

**Wide Door Loop Termination**: `MakeWideDoorVisible()` is called for multi-tile doors, but its implementation is not visible. If it modifies `tilemap[][]` during casting, there could be race conditions or visibility order dependencies.

**Interpolation Assumes Identical Posttype/Lump**: `Interpolate()` copies `posts[x1].posttype` and `posts[x1].lump` to all intermediate posts. If the wall type changes mid-interpolation (e.g., a window followed by a door), the intermediate posts will be wrong. This is a hidden assumption about spatial coherence.
