# rott/texture.c — Enhanced Analysis

## Architectural Role
This file implements the **perspective-corrected texture mapping stage** of the 3D rendering pipeline. It bridges scan conversion (which produces edge-linked scanlines in `Scanline[]`) and low-level rasterization (via `TextureLine()`). `XDominantFill()` computes per-segment texture coordinate gradients using one-over-w perspective correction, then delegates actual pixel fill to the hardware/asm layer. This two-stage design separates high-level geometry math from tight inner-loop rasterization.

## Key Cross-References

### Incoming (who calls `XDominantFill`)
- **Rendering pipeline**: Likely invoked from `rt_draw.c` or `rt_build.c` after scan conversion completes and before frame buffer output
- **Scanline data source**: Depends on `Scanline[]` global array (populated by preceding scan conversion stage)
- **Display buffer**: Writes to shared `display_buffer` (frame buffer)

### Outgoing (what this file depends on)
- **`PreprocessScanlines()`**: Scanline preprocessing (edge clipping, sentinel setup) — likely defined in `rt_build.c` or related module
- **`TextureLine()`**: Low-level texture rasterizer (likely asm or inline) — processes one fill segment given configured globals
- **Fixed-point math**: `FixedDiv2()`, `FixedMul()` macros (likely in a math header)
- **Global rendering state**: 
  - Read: `Scanline[]`, `_minscanline`, `_maxscanline`, `Xmax`, `_texture`, `BufferPosition[]`, `display_buffer`
  - Write: `texture_*` globals (parameter passthrough to `TextureLine()`)

## Design Patterns & Rationale

**Parameter Passing via Globals**: Rather than passing 8+ texture parameters as function arguments, the function sets module-level globals (`texture_u`, `texture_v`, etc.) before each `TextureLine()` call. This was idiomatic in 1990s tight-loop rendering to reduce stack/register pressure.

**Perspective-Correct Interpolation (1/w method)**: 
- Computes `1/w` at scanline edges, then multiplies `u` and `v` by it
- Derives linear gradients (`du`, `dv`) in *perspective space* along the scanline
- `TextureLine()` then linearly steps through texture coordinates, which accounts for perspective warping
- **Why**: Direct linear interpolation of `u` and `v` in 2D screen space causes texture distortion; this fixes it cheaply without per-pixel division.

**Linked-List Scanlines**: `Scanline[]` is an array of sentinel-headed doubly-linked lists (one per row). The loop skips empty segments (`if (Scanline[i] == Scanline[i]->next) continue`). This allows efficient sparse scanline representation for non-convex polygons.

## Data Flow Through This File

1. **Input**: `Scanline[]` (doubly-linked segment lists, one per scanline)
2. **Per-scanline**:
   - Read perspective-corrected (u/w, v/w) at left and right edges
   - Compute linear gradients (du, dv) across scanline width
   - Initialize (u, v) at left edge
3. **Per-segment** (inner loop):
   - Load segment pixel count and boundaries
   - Configure texture globals (`texture_u`, `texture_v`, `texture_du`, `texture_dv`, `texture_count`, `texture_dest`, etc.)
   - Call `TextureLine()` to fill pixels
   - `TextureLine()` increments (u, v) internally by (du, dv) per pixel and indexes `_texture[]`, writing to `display_buffer`
4. **Output**: Framebuffer filled with textured pixels

## Learning Notes

**Fixed-Point Arithmetic**: The entire math uses fixed-point (`fixed` type) to avoid floating-point overhead on 1990s CPUs. Note `dx = ((endx - startx) << 16)` — the shift encodes sub-pixel precision.

**Perspective Correction Insight**: Modern engines use per-pixel 1/z in framebuffer (depth buffer) or compute it in shaders. Here, perspective is baked into gradient computation upfront — simpler but less flexible for curved surfaces.

**Scanline Algorithm Lineage**: This traces to Gouraud shading and early texture mappers (pre-3D hardware). The scanline-based approach is now obsolete (rasterizers handle it), but studying it teaches projection math, edge walking, and interpolation.

**Tight Loop Separation**: `TextureLine()` is separate because it's likely hand-optimized asm or inline code that must be extremely fast (called per segment, per frame). Keeping parameter setup in C keeps the code readable.

## Potential Issues

- **Loop bounds uncertainty**: Loop iterates `_numscanlines + 3` with offset initialized at `_minscanline - 2`. The `+3` and `-2` offsets suggest guard scanlines for clipping; unclear if this is always safe without seeing the scanline setup code.
- **Global state coupling**: Texture globals are written repeatedly without synchronization or validation. If `TextureLine()` is interrupted or if another thread reads these, corruption could occur. Likely not a problem in single-threaded 1990s context but a code smell by modern standards.
- **Silent division by zero**: If `dx == 0` (all pixels in segment have same x), `du` and `dv` are zeroed silently. This is probably intentional (vertical segments get no gradient) but masks potential bugs if scanline data is malformed.
