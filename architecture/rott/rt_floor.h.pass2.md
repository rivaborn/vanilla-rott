# rott/rt_floor.h — Enhanced Analysis

## Architectural Role

This header exposes the **floor/ceiling plane rendering subsystem**, a core component of the raycaster's per-frame rendering pipeline. It sits downstream of ray-casting core (which populates `mr_xstep`, `mr_ystep`, `mr_xfrac`, `mr_yfrac`) and feeds into the main render loop. The `sky` flag couples the plane renderer to game logic, allowing dynamic control over parallax sky rendering.

## Key Cross-References

### Incoming (who depends on this file)
- **Main render loop** calls `DrawPlanes()` once per frame (after screen clear, before or after wall rendering)
- **Engine initialization** calls `SetPlaneViewSize()` on startup or resolution change
- **Game logic** reads/writes `sky` flag to toggle parallax mode on/off
- **Sky tile generation system** uses `MakeSkyTile()` to populate sky texture cache
- **Scene composition** likely calls `DrawFullSky()` and checks `SkyExists()` conditionally

### Outgoing (what this file depends on)
- **Ray-casting core** sets `mr_xstep`, `mr_ystep`, `mr_xfrac`, `mr_yfrac` before each plane draw
- **Video memory/framebuffer** (implicit; modified by `DrawPlanes()` and `DrawFullSky()`)
- **Texture/tile system** (implicit; `MakeSkyTile()` likely reads from asset cache)

## Design Patterns & Rationale

**Global Ray-Marching State**: The `mr_*` globals expose internal stepping values used during plane rasterization. This is a typical 1990s fixed-pipeline pattern—avoids function call overhead and allows tight assembly loops. Modern engines would pass this via structured state or compute buffers.

**Dual Sky API**: `MakeSkyTile()` + `DrawFullSky()` suggests **lazy/dynamic sky generation**—tiles are created on-demand (possibly at level load or cache miss) rather than pre-baked. This was common to conserve 16-bit era memory.

**Boolean Sky Flag**: Treating parallax sky as a simple `int` flag rather than an enum indicates it's a **runtime toggle**, not a compile-time feature. Implies sky can be enabled/disabled per-level or via cheat code.

## Data Flow Through This File

```
Ray-casting core
    ↓ (writes mr_xstep, mr_ystep, mr_xfrac, mr_yfrac)
DrawPlanes()
    ↓ (uses globals to march rays through floor/ceiling)
Framebuffer

Game logic
    ↓ (sets sky flag)
DrawFullSky() / SkyExists()
    ↓ (may call MakeSkyTile internally)
Framebuffer / Sky texture cache
```

## Learning Notes

- **Idiomatic to era**: This is how Wolfenstein 3D and early Doom handled planes—a separate rasterizer path for floors/ceilings, distinct from wall-column rendering.
- **No state encapsulation**: Direct global variables for stepping values. Modern engines hide this behind viewport/camera state objects.
- **Procedural sky**: `MakeSkyTile()` taking a raw `byte*` suggests runtime sky generation (e.g., gradient, pattern, or palette lookup), not just blitting a pre-rendered image. This was a memory optimization.
- **Separation of concerns**: Sky logic is cleanly separated (`SkyExists`, `MakeSkyTile`, `DrawFullSky`) even though parallax sky might be optional.

## Potential Issues

- **Uninitialized globals**: `mr_*` variables must be set before `DrawPlanes()` is called; no assertions or guards against stale values.
- **Raw pointer in `MakeSkyTile(byte *tile)`**: No size validation; caller must know tile buffer size.
- **Race condition risk**: If `sky` flag is toggled mid-render, undefined behavior; assumes single-threaded frame loop.
- **No viewport validation**: `SetPlaneViewSize()` has no return value; silent failure if viewport is invalid.
