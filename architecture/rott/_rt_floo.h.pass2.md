# rott/_rt_floo.h — Enhanced Analysis

## Architectural Role
This header defines the compile-time constraints for the raycasting renderer's view and sky geometry systems. As part of the "ray-traced" (`_rt_*`) subsystem, it establishes hard limits that directly impact memory allocation, rendering loop bounds, and culling decisions in the software raycaster's main render pipeline. The constants are tightly coupled to the fixed-screen rendering model typical of early-90s DOS raycasters (Wolfenstein 3D lineage).

## Key Cross-References

### Incoming (who depends on this file)
- **Rendering pipeline files** (rt_draw.c, rt_view.c) — likely reference MAXVIEWHEIGHT for scanline/column loop bounds
- **Memory allocation subsystems** — use MAXSKYSEGS and MAXSKYDATA to allocate geometry buffers at startup
- **Sky rendering module** (inferred from `rt_sky` or similar) — uses MAXSKYSEGS to bound segment iteration, MINSKYHEIGHT for horizon/culling logic

### Outgoing (what this file depends on)
- **Global screen configuration** — MAXVIEWHEIGHT aliases MAXSCREENHEIGHT (defined elsewhere, likely in `rt_const.h` or similar global header)
- No runtime dependencies; operates purely at compile-time

## Design Patterns & Rationale
**Hard-limit pattern**: All four constants are fixed at compile-time, preventing dynamic buffer resizing. This is characteristic of DOS-era engines with fixed memory pools.

- **MAXVIEWHEIGHT = MAXSCREENHEIGHT** — Ensures consistency between screen resolution and render viewport (likely 320×200 or 320×240)
- **MAXSKYSEGS = 2048** — Conservative upper bound on sky geometry that won't thrash cache in inner rendering loop
- **MAXSKYDATA = 8** — Suggests each sky segment carries ~8 data elements (possibly: angle, height, color, texture coords, etc.)
- **MINSKYHEIGHT = 148** — This pixel threshold is unusual; in classic raycasters, the horizon is often fixed at screen midpoint (~100px in 320×200). 148 pixels suggests either a non-standard viewport ratio or a culling threshold to skip rendering when sky ceiling is too low.

## Data Flow Through This File
No runtime data flow; this header is processed **once at compile-time**:
1. Included by rendering modules during compilation
2. Constants expand inline wherever sky/floor rendering loops iterate
3. Memory allocator(s) pre-allocate geometry buffers sized by MAXSKYSEGS × MAXSKYDATA

## Learning Notes
**Idiomatic to this era/engine:**
- Absence of dynamic resizing — typical of fixed-memory DOS engines; contrasts with modern engines (Unreal, Unity) that allocate per-frame or per-object
- Hard-coded viewport constraints reflect the 320×200 VGA standard of mid-90s (ROTT released 1995)
- The `_rt_` naming suggests this was part of a "ray-traced" subsystem rename/refactor at some point (likely to distinguish from other rendering paths)

**Game engine concepts:**
- **View constraints** — MAXVIEWHEIGHT enforces the rendering contract; any code assuming larger views will silently overflow
- **Geometry culling** — MINSKYHEIGHT is an early culling gate; levels with low ceilings skip expensive sky rendering

## Potential Issues
- **Tight coupling to MAXSCREENHEIGHT**: If screen resolution is changed dynamically (e.g., for windowed mode), MAXVIEWHEIGHT won't adapt, potentially causing buffer overruns in view rendering loops. Only recoverable if MAXSCREENHEIGHT is also defined as a compile-time constant (likely is, but not visible here).
- **No safety margin on MAXSKYSEGS**: If geometry generation ever creates 2048+ segments, the geometry builder will silently drop or corrupt the overflow. Modern engines would either error at load-time or dynamically realloc.
- **MINSKYHEIGHT magic number**: 148 is unexplained; if ceiling heights in maps are adjusted or new level formats introduced, this culling threshold may become incorrect, silently disabling sky rendering in short corridors.
