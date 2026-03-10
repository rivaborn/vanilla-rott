# rott/rt_dr_a.asm — Enhanced Analysis

## Architectural Role

This file is the **low-level raster rendering engine** for ROTT's raycasted 3D view. It sits at the bottom of the rendering pipeline, receiving pre-computed parameters (viewport state, texture columns, shading tables) and performing pixel-perfect VGA writes. The four post-drawing variants suggest multiple rendering contexts: the main 3D view (DrawPost_, DrawHeightPost_), UI/HUD menus (DrawMenuPost_), and the minimap (DrawMapPost_). RefreshClear_ and SetMode240_ form the frame-lifecycle bookends—mode setup at init, buffer clear each frame.

## Key Cross-References

### Incoming (who depends on this file)
- **Rendering pipeline (rt_draw.c):** Likely calls `SetMode240_()` once during initialization and `RefreshClear_()` each frame before rendering. The `BuildTables()` and `CalcHeight()` functions from rt_draw.h prepare parameters that feed into the post-drawing routines.
- **Viewport system (rt_view.c):** `CalcProjection()` and `ChangeFocalWidth()` compute the fractional scaling parameters (`_hp_startfrac`, `_hp_srcstep`) that these functions consume.
- **Actor/game logic (rt_actor.c, rt_playr.c):** Indirectly—they maintain state (_viewwidth, _viewheight, _centery, _bufferofs) that these functions read to position and dimension the 3D view.

### Outgoing (what this file depends on)
- **Global state:** _spotvis (128×128 visibility grid), _viewwidth/height (viewport dimensions), _bufferofs (framebuffer base), _fandc (flag controlling ceiling/floor clearing), _centery (horizon line), _shadingtable (palette lookups), _hp_startfrac/_hp_srcstep (fractional scaling).
- **Hardware:** VGA I/O ports (sequencer, CRT controller, MISC output) for mode-setting; linear framebuffer at 0xa0000 for pixel writes.

## Design Patterns & Rationale

**Self-modifying code:** `DrawHeightPost_`, `DrawMenuPost_` patch their own loop instructions (hp1/hp2, mhp1/mhp2) with the source step increment. This avoids passing `_hp_srcstep` as a loop-invariant—a 1990s micro-optimization common in performance-critical assembly. Modern CPUs and branch predictors make this pattern obsolete.

**Dual-pixel loop unrolling:** `DrawHeightPost_` renders two pixels per iteration, interleaving their fractional calculations to hide dependency latency on x86-32. Each pixel tracks its own fraction (ebp, edx) in parallel before writing.

**Lookup-table shading:** Colors come pre-shaded via `_shadingtable`—the pixel value indexes directly into a palette, eliminating expensive per-pixel lighting math. This is the classic "color translate" pattern used in id-tech engines.

**Code variants for context:** Three post-drawing functions with identical structure but different shading paths reflect the engine's multi-context rendering: DrawHeightPost_ (main 3D, full shading), DrawMenuPost_ (menu, no shading), DrawMapPost_ (map, unscaled source). This avoids branch mispredictions in the tight inner loop.

## Data Flow Through This File

**Initialization:**
1. `SetMode240_()` runs once: reprogram VGA sequencer and CRT timing registers, clear VRAM
2. `RefreshClear_()` early each frame: zero visibility array and paint ceiling/floor regions

**Per-frame rendering (main view):**
1. Raycaster determines visible wall segments and their column coordinates
2. For each segment: `DrawHeightPost_()` or `DrawPost_()` called with:
   - ESI = source texture column (width 64, height 32)
   - EDI = screen offset (pre-computed via `_ylookup`)
   - ECX = post height in pixels; `_hp_srcstep` / `_hp_startfrac` pre-loaded globally
3. Function reads source pixels via fixed-point fraction in EBP, shades via `_shadingtable`, writes to framebuffer
4. Menu/map rendering: `DrawMenuPost_()` / `DrawMapPost_()` bypass shading, write raw or unscaled source pixels

**Key insight:** Caller pre-computes *everything* except the per-pixel source indexing and shading lookup—the function does minimal work per pixel (one `shr`+`mov` for fraction, one table lookup, one write).

## Learning Notes

**Engine-era idioms absent in modern engines:**
- **Self-modifying code** was a legitimate micro-optimization in the 1990s; modern CPUs punish it (pipeline flushes).
- **Lookup-table shading** avoids fragment shader calls; modern GPUs make this pattern irrelevant.
- **VGA programming** (sequencer/CRT registers) is DOS/Win95 legacy; today's rasterizers target GPU command streams or graphics APIs.
- **Manual palette management** (256-color lookup tables) replaced by true-color framebuffers.

**Connections to game engine concepts:**
- This is a **scanline renderer** for a raycasted view—similar to Wolfenstein 3D's approach.
- The fractional scaling (EBP as 16.16 fixed-point) is a **texture-mapping technique** predating per-pixel interpolation.
- `_shadingtable` is a **light-accumulation pattern**: baked light is encoded in the palette, not computed per pixel.

**Key takeaway for a developer:** This file represents the **innermost critical path** of the engine—every visible wall pixel flows through one of these four functions. The design prioritizes CPU cache efficiency (tight loops, pre-computed tables) and avoids branches in the hot path.

## Potential Issues

1. **Self-modifying code fragility:** If `DrawHeightPost_()` is called without setting `_hp_srcstep` first, garbage is written to hp1/hp2 patches. No guard against this.
2. **Hardcoded screen width (SCREENBWIDE=96):** The margin calculation `sub edx, eax` assumes a fixed 320-pixel logical width. If `_viewwidth` is set dynamically, this breaks.
3. **Viewport center (`_centery`) bounds checking:** `RefreshClear_()` clamps ceiling region to viewheight if centery > viewheight, but doesn't validate centery >= 0. Negative centery may cause underflow in row pointer arithmetic.
4. **No visibility clipping in post drawing:** `DrawPost_()` limits drawn height to viewport height but doesn't clip horizontally. Caller must ensure ESI doesn't overrun the source texture.
