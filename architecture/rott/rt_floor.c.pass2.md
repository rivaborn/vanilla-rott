# rott/rt_floor.c — Enhanced Analysis

## Architectural Role

**rt_floor.c** is the VGA-level rasterizer for horizontal planes (floor and ceiling) in Rise of the Triad's software raycaster, sitting between the high-level frame rendering pipeline and low-level hardware. After wall edges are rasterized (by rt_draw/rt_view), this module fills the remaining gaps with textured floor/ceiling spans or parallax sky, using fixed-point texture mapping and chunked VGA writes. It's a critical bottleneck: every visible pixel below the horizon and above floor geometry flows through either `DrawPlanes()` or `DrawFullSky()`.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_draw.c** or **rt_view.c**: Calls `DrawPlanes()` after wall rasterization in the main frame-render loop; likely called once per frame
- **rt_main.c**: Calls `SetPlaneViewSize()` at level load to initialize floor/ceiling/sky textures
- **rt_actor.c** / **rt_playr.c**: Indirectly affected—floor/ceiling clips stored in `posts[]` array drive visibility culling here

### Outgoing (what this file depends on)
- **w_wad.h** (`W_CacheLumpNum`, `W_GetNumForName`): Resource loading for floor, ceiling, and sky texture lumps (FLRCL1–FLRCL16, SKYSTART+offset)
- **z_zone.h** (`SafeMalloc`, `SafeFree`): Heap allocation for merged sky data buffer (256×400 bytes)
- **rt_fc_a.asm** (`DrawSkyPost`, `DrawRow`): Assembly-language tight loops for per-plane rasterization
- **modexlib.h** (`VGAWRITEMAP`, `VGAMAPMASK`): Direct VGA register I/O for plane selection
- **rt_view.h** / **rt_draw.h** globals: `posts[]` (wall ceiling/floor clips), `pixelangle[]`, `viewangle`, `colormap`, `shadingtable`, `bufferofs`, `ylookup[]`, `lightninglevel`, `fog`, `fulllight`, `basemaxshade`
- **engine.c**: Accesses `player` struct (z position, viewing parameters)
- **rt_def.h** / **_rt_floo.h**: Constants (`MAXVIEWHEIGHT`, `MAXSKYSEGS`, `FINEANGLES`, `MINSKYHEIGHT`)

## Design Patterns & Rationale

**VGA Planar Graphics Model:**  
The engine splits the 320×200 framebuffer into 4 interleaved bitplanes (one per CPU register write). This halves memory bandwidth but requires four separate passes per scanline. `DrawHLine()` and `DrawSkyPost()` exploit this: iterate `dest += 4` and write via `VGAWRITEMAP(plane)` to touch all pixels with minimal cache thrashing. Classic early-90s optimization for 286/386 CPUs with slow DRAM.

**Fixed-Point Texture Coordinates:**  
`mr_xfrac` / `mr_yfrac` and stepping values `mr_xstep` / `mr_ystep` use fixed-point (16-bit integer + 16-bit fraction) to approximate perspective-correct texture mapping without floating-point division per pixel. Precomputed in `DrawHLine()` from view parameters.

**Parallax Sky as Sprite Overlay:**  
Sky is not baked into the floor texture but rendered separately with horizon-height parameterization. Map defines sky type (MAPSPOT 1,0,0 ≥ 234) and horizon sprite (MAPSPOT 1,0,1). This allows per-map sky customization without texture duplication.

**Deferred Lighting:**  
`SetFCLightLevel()` selects a pre-computed shading colormap row based on distance (`height`), rather than per-pixel lighting. Enables fast lookup shading but restricts visual variety—all pixels at the same distance have the same shade.

## Data Flow Through This File

```
Level Load:
  SetPlaneViewSize() → reads MAPSPOT for sky/floor/ceiling IDs
    → loads lumps via W_CacheLumpNum
    → merges sky textures (MakeSkyData) into unified buffer
    → precomputes skysegs[] lookup table (256 sky columns, rotated by angle)

Per Frame:
  DrawPlanes() called with walls already rasterized
    → if sky != 0: DrawSky() fills ceiling area with parallax
    → else: iterates horizontally, collecting floor/ceiling spans
      → for each span: DrawHLine() calculates texture UV steps
        → dispatches assembly DrawRow() calls (one per VGA plane)
        → DrawRow() writes textured pixels to bufferofs

Texture Mapping Sequence (DrawHLine):
  1. Compute distance from player to scanline (fixed-point)
  2. Calculate texture U,V step rates (FixedMulShift from viewsin/viewcos)
  3. Set shadingtable based on distance + fog/lightning
  4. Store U,V, destination, step count in globals (mr_*)
  5. Assembly DrawRow() reads globals, writes 4 pixels per loop iteration
```

## Learning Notes

**Era-Specific Constraints:**  
This code exemplifies 1994 DOS raycaster architecture:
- No dynamic lighting; shading is distance-based LUT only
- VGA planar writes (4 separate passes) instead of linear framebuffer
- Assembly inner loops mandatory for ~60 FPS at 320×200 with software rasterization
- Textures must fit in conventional memory; lump-based streaming would be too slow

**vs. Modern Engines:**
- Modern deferred rendering separates lighting from geometry; this bakes it into colormaps
- Modern GPUs handle 4-plane chunking transparently; VGA register mucking is explicit here
- Modern engines use floating-point perspective-correction; this uses fixed-point lookup tables
- Sky is typically a cubemap or shader; here it's a precomputed rotation table

**Idiomatic Patterns:**
- **Global render state** (`mr_*` variables): Used to avoid function call overhead in tight assembly loops—each `DrawRow()` invocation reads these globals rather than taking parameters
- **Lump-based resource loading:** Files are packed into a WAD archive; level content defined via sprite IDs in map layer 1

## Potential Issues

1. **Hard-coded Lump Variants:**  
   `GetFloorCeilingLump()` supports only 16 floor/ceiling pairs (FLRCL1–FLRCL16). Exceeding this limit errors; no fallback or streaming.

2. **Fragile Horizon Parameterization:**  
   `SetPlaneViewSize()` expects specific sprite icons at (1,0,1) on map plane 1 to encode horizon height. Sprite IDs 90–97 map to heights 1–8; 450–457 map to 9–16. If a level designer misses this sprite or uses wrong ID, `Error()` fires. No graceful degradation or default.

3. **VGA Hardware Coupling:**  
   All rendering paths assume 256-color Mode X VGA. Porting to linear framebuffer or 16-bit color requires rewriting `DrawRow()` and `DrawSkyPost()` assembly routines.

4. **Sky Data Leakage:**  
   If `SetPlaneViewSize()` is called twice in sequence without proper cleanup, `oldsky > 0` check should free the old buffer. However, if level transitions fail or abort prematurely, `SafeFree()` might not fire, leaking 256×400 bytes per level.

5. **Perspective Correction Approximation:**  
   Fixed-point stepping assumes linear interpolation is "good enough." On distant scanlines (high yp), texture distortion becomes visible; true perspective-correct would require division per pixel or higher-resolution lookup tables.
