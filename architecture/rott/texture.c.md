# rott/texture.c

## File Purpose
Implements texture-mapped rasterization for horizontal (X-dominant) scan segments. The `XDominantFill()` function processes scanline data and drives per-segment texture fill operations by computing perspective-corrected texture coordinate gradients and invoking the low-level texture rasterizer.

## Core Responsibilities
- Compute texture coordinate interpolation parameters (du/dv) along scanlines
- Iterate over active scan segments in each scanline
- Calculate perspective-corrected texture coordinates using fixed-point math
- Configure global texture state and invoke `TextureLine()` for hardware/low-level rasterization
- Manage scanline preprocessing and bounds tracking

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| scan_t | struct (inferred) | Scanline segment node with linked-list structure; holds u, v, w (perspective), value (x-coord), and next/prev pointers |
| fixed | typedef (inferred) | Fixed-point numeric type for sub-pixel precision in texture math |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| texture_u | int32 | static | Current texture u-coordinate for active segment |
| texture_v | int32 | static | Current texture v-coordinate for active segment |
| texture_count | int32 | static | Pixel count for current segment |
| texture_du | int32 | static | Texture u-coordinate delta per pixel |
| texture_dv | int32 | static | Texture v-coordinate delta per pixel |
| texture_source | byte* | static | Pointer to source texture bitmap |
| texture_dest | byte* | static | Pointer to destination framebuffer region |
| texture_destincr | byte* | static | Pointer into buffer position lookup table |

## Key Functions / Methods

### XDominantFill
- **Signature:** `void XDominantFill(void)`
- **Purpose:** Rasterize textured scan segments for all active scanlines via perspective-corrected interpolation.
- **Inputs:** 
  - Global state: `Scanline[]` (linked-list array), `_minscanline`, `_maxscanline`, `Xmax`, `_texture`, `BufferPosition`, `display_buffer`
- **Outputs/Return:** None; modifies framebuffer via `TextureLine()` calls.
- **Side effects:** 
  - Calls `PreprocessScanlines()` (scanline preprocessing)
  - Calls `TextureLine()` for each active segment (fills framebuffer)
  - Modifies all `texture_*` globals
- **Calls:** `PreprocessScanlines()`, `FixedDiv2()`, `FixedMul()`, `TextureLine()`
- **Notes:** 
  - Assumes `Scanline[i]` is a sentinel-headed doubly-linked list; skips empty lists.
  - Uses one-over-w (reciprocal) perspective correction: `u/w` and `v/w` are computed at scanline edges, then interpolated linearly.
  - Fixed-point math: `dx` is shifted left by 16 bits; du/dv use `FixedDiv2()`.
  - Loop iterates over `_numscanlines + 3` scanlines; offset increments by `Xmax` each iteration.

## Control Flow Notes
This function is invoked during the **render phase** after 3D scan conversion:
1. Scanline preprocessing (edge setup, clipping)
2. Per-scanline loop: compute perspective gradients at top-left corner
3. Inner loop: iterate over all fill segments, configure texture state, and invoke `TextureLine()` to rasterize
4. Framebuffer is filled progressively; output appears after all calls complete

## External Dependencies
- **Preprocessing:** `PreprocessScanlines()` (defined elsewhere)
- **Texture math:** `FixedDiv2()`, `FixedMul()` (fixed-point operations, likely macros or inlined)
- **Rasterization:** `TextureLine()` (low-level texture fill, defined elsewhere)
- **Global data:** `Scanline[]`, `_minscanline`, `_maxscanline`, `Xmax`, `_texture`, `BufferPosition`, `display_buffer` (all defined elsewhere; likely in a shared rendering context)
