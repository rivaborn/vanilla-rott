# rott/rt_vid.h — Enhanced Analysis

## Architectural Role

This header defines the rendering interface for a software-based, tile-oriented 2D graphics engine. It sits at the boundary between game logic (RT_GAME, RT_ACTOR, RT_MENU) and low-level VGA hardware access, providing a two-tier drawing model: immediate-mode primitives (VL_*) for direct screen writes and buffered operations (VWB_*) for dirty-rect optimization. The tile system (PORTTILESWIDE=20, PORTTILESHIGH=13) strongly suggests a fixed 320×200 viewport resolution typical of 1990s VGA modes, with DrawTiledRegion handling tilemap rendering for the game world.

## Key Cross-References

### Incoming (who depends on this file)
- **Game rendering loop** (RT_DRAW.C, RT_GAME.C): Calls VW_MarkUpdateBlock and VW_UpdateScreen to coordinate frame updates
- **Menu system** (RT_MENU.C, RT_CONTROL.C): Uses MenuFadeOut/MenuFadeIn macros, VL_FadeOut/VL_FadeIn for UI transitions
- **Cinematic/script system** (CIN_*.C): Calls VL_DecompressLBM to load and display cutscene graphics with embedded palettes
- **HUD/text rendering** (RT_TEXT.C, RT_DRAW.C): Uses VWB_DrawPic and VWB_Bar for menu backgrounds, dialogs, and UI elements
- **Game world view** (RT_DRAW.C): Core consumer of DrawTiledRegion for rendering the main viewport tilemap

### Outgoing (what this file depends on)
- **lumpy.h**: Type definitions (pic_t, lbm_t, patch_t, font_t); defines embedded palette structure in lbm_t
- **VGA hardware** (implicit): Direct access to framebuffer memory and DAC for palette operations; screenfaded global suggests coordination with fade hardware state
- **Memory subsystem**: Manages external pointers (updateptr) and preallocated buffers (update[], blockstarts[])

## Design Patterns & Rationale

1. **Buffered vs. Immediate Drawing**: VL_* functions bypass buffering for low-level control (e.g., palette fades); VWB_* functions write through the dirty-rect buffer for batch screen updates. This dual API allows both fine-grained control and frame coherence.

2. **Dirty-Rect Optimization**: Rather than re-rendering the entire 320×200 frame, the engine marks rectangular regions for update (VW_MarkUpdateBlock) and batches writes into a 20×13 block grid (UPDATESIZE=260 bytes). This was critical on early VGA cards with slow bus bandwidth.

3. **Tile-Based Rendering**: DrawTiledRegion with offset parameters (offx, offy) enables texture tiling with sub-tile scrolling, typical of early raycasters and tile engines.

4. **Palette as State**: The screenfaded boolean couples palette animation state to rendering, allowing the game loop to coordinate fade effects without polling hardware.

5. **Precomputed Lookup Tables**: mapwidthtable[], uwidthtable[], blockstarts[] pre-baked in advance to avoid per-frame division/multiplication overhead.

**Rationale**: DOS-era optimization constraints (slow CPU, slow memory, slow I/O) drive these patterns. Modern engines use framebuffer blitting and GPU, making dirty-rect tracking unnecessary.

## Data Flow Through This File

```
Game Logic
  ↓
[Mark regions] VW_MarkUpdateBlock (x1, y1, x2, y2)
  → Sets blockstarts[] entries in update[] buffer
  ↓
[Render] VWB_DrawPic, DrawTiledRegion, VWB_Bar
  → Writes to updateptr (internal buffer memory)
  ↓
[Flush] VW_UpdateScreen
  → Reads marked blocks from update[] buffer
  → Copies each block's pixels to VGA framebuffer
  ↓
Palette changes:
  VL_SetColor → DAC hardware
  VL_FadeOut/VL_FadeIn → Interpolate palette, write all 256 entries
  screenfaded ← synchronized state flag
```

The separation of marking (VW_MarkUpdateBlock) from rendering to flushing (VW_UpdateScreen) decouples game logic from screen synchronization, enabling flexible frame timing.

## Learning Notes

1. **Pre-3D Tile Engine**: This is a 2D-only engine; all perspective/3D rendering occurs elsewhere (likely RT_VIEW.C for raycasting). rt_vid.h handles UI, HUD, and tilemap background layers.

2. **Double-Buffering via Dirty Rects**: Instead of a full backbuffer (wasteful in 1990s), the engine uses a sparse update list, trading memory for CPU cost. Contrast with modern triple-buffering (unlimited VRAM).

3. **Hardware Palette as Game State**: The 256-color VGA palette is a global resource; fade effects coordinate via screenfaded. Modern engines use per-sprite/per-layer blending, avoiding global palette conflicts.

4. **Tile Grid Resolution**: 20×13 tiles for a 320×200 screen suggests 16×16-pixel tiles (or similar). This is a hard constraint; resizing would require recomputing blockstarts[].

5. **Separation of Concerns**: VL_* = video/hardware primitives; VWB_* = video windowed/buffered; this naming convention is consistent across the codebase and aids code navigation.

## Potential Issues

1. **Global Dirty-Rect Buffer Coupling**: The update[] buffer is global and externally managed (updateptr, blockstarts[] allocated elsewhere). Reentrancy or nested update regions could cause corruption.

2. **No Error Handling**: All drawing functions return void or int (status), but callers likely don't check return codes. Palette fade failures or LBM decompress errors would silently fail.

3. **Fixed Tile Resolution**: PORTTILESWIDE and PORTTILESHIGH are compile-time constants. Dynamic resolution changes (e.g., windowed mode) would require recompilation.

4. **Palette Invariants Not Enforced**: VL_FadeOut/In assume screenfaded is managed externally and the palette is in a known state; no validation that prevents overlapping fades or palette corruption.

5. **LBM Flip Parameter Underspecified**: VL_DecompressLBM(lbm_t *lbminfo, boolean flip) lacks documentation of what "flip" does (vertical? horizontal?) or how it interacts with lbm_t structure format.
