# rott/rt_sc_a.asm

## File Purpose
Low-level x86-32 assembly implementation of column rasterization routines for raycasting-based 3D rendering. Contains five specialized column drawing functions that perform texture mapping, color translation, and pixel writes to the framebuffer with different visual effects (textured, solid, transparent, clipped, high-precision wall).

## Core Responsibilities
- Rasterize vertical columns from texture sources to screen memory with fixed-point texture coordinate interpolation
- Apply color translation via shading lookup tables for lighting/palette effects
- Optimize rendering via pair-processing (2 pixels per loop iteration) and self-modifying code
- Support multiple rendering variants: textured, solid-fill, transparent, clipped bounds, and high-precision wall textures
- Manage screen address calculation and vertical iteration

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| loopcount | DWORD | static | Loop iteration counter (pixel pairs) |
| pixelcount | DWORD | static | Total pixels to render in column |
| SCREENWIDTH | constant | file-static | Screen stride = 96 pixels |

## Key Functions / Methods

### R_DrawColumn_
- **Signature:** `void R_DrawColumn_(void)`  
- **Purpose:** Render a textured column with fixed-point texture mapping and color translation.
- **Inputs:** edi = screen column base address (pre-loaded); external globals: _dc_yl (Y low), _dc_yh (Y high), _dc_iscale (inverse texture scale), _dc_texturemid (texture offset), _dc_source (texture data), _shadingtable (color LUT), _centery (camera Y).
- **Outputs/Return:** Pixels written to framebuffer at edi.
- **Side effects:** Modifies screen memory; patches instructions at patch1 and patch2 with scaled increment value.
- **Calls:** None (all operations inline).
- **Notes:** Uses pair-processing (2 pixels/iteration). Fixed-point texture coords shifted right 16 bits for index. Self-modifying code to set _dc_iscale into add instructions.

### R_DrawSolidColumn_
- **Signature:** `void R_DrawSolidColumn_(void)`  
- **Purpose:** Fill a column with a single color (no texture mapping).
- **Inputs:** edi = screen column base; bl = color value; _dc_yl, _dc_yh (bounds).
- **Outputs/Return:** Solid-color pixels written to framebuffer.
- **Side effects:** Modifies screen memory.
- **Notes:** Fastest variant—pure vertical fill with no lookups.

### R_TransColumn_
- **Signature:** `void R_TransColumn_(void)`  
- **Purpose:** Render a transparent/translucent column by reading existing screen pixels and remapping through shading table.
- **Inputs:** edi = screen column base; _dc_yl, _dc_yh (bounds), _shadingtable (transparency LUT).
- **Outputs/Return:** Translated pixels written back to framebuffer.
- **Side effects:** Reads and overwrites screen memory.
- **Notes:** Implements alpha-blending equivalent via color table lookup.

### R_DrawClippedColumn_
- **Signature:** `void R_DrawClippedColumn_(void)`  
- **Purpose:** Textured column rendering with clipped vertical bounds (uses _centeryclipped instead of _centery).
- **Inputs:** Same as R_DrawColumn_ except _centeryclipped (clipped camera Y) and _dc_centeryclipped.
- **Outputs/Return:** Clipped textured pixels to framebuffer.
- **Side effects:** Modifies screen and patches apatch1/apatch2.
- **Notes:** Nearly identical to R_DrawColumn_ but with clipping support.

### R_DrawWallColumn_
- **Signature:** `void R_DrawWallColumn_(void)`  
- **Purpose:** High-precision wall texture rendering using 10-bit fixed-point shifts for improved accuracy.
- **Inputs:** edi = screen column base; _dc_yl, _dc_yh, _dc_iscale, _dc_texturemid, _dc_source, _shadingtable, _centery.
- **Outputs/Return:** High-precision textured pixels to framebuffer.
- **Side effects:** Modifies screen and patches wcpatch1/wcpatch2 (10-bit shifted increment).
- **Notes:** Texture coords shifted left 10 bits, then right 26 bits (net 16-bit) for finer granularity. Commented-out masking suggests potential optimization path.

## Control Flow Notes
Each function follows a similar pattern:
1. **Initialization:** Load bounds (_dc_yl, _dc_yh) and calculate pixel count; compute initial texture coordinate (ebp = _dc_texturemid - scale * camera offset).
2. **Self-modification:** Patch increment constants into add instructions.
3. **Main loop:** Pair-processing loop—calculate two texture indices in parallel, fetch source pixels, translate via shading table, write to screen stride SCREENWIDTH apart.
4. **Tail:** Check for odd final pixel and write if needed.
5. **Exit:** Restore ebp (if pushed) and return.

Fits into **render phase** of frame: called once per visible column from raycasting loop.

## External Dependencies
- **Includes/defines:** `.386`, `.MODEL flat` (flat memory model, 32-bit addressing).
- **External symbols referenced:**
  - _centery, _centeryclipped (DWORD) — camera Y position
  - _dc_yl, _dc_yh (DWORD) — column Y bounds
  - _dc_iscale (DWORD) — inverse scale for texture coordinate step
  - _dc_texturemid (DWORD) — base texture coordinate offset
  - _ylookup (DWORD array) — Y-to-screen-offset LUT
  - _dc_source (DWORD) — texture bitmap pointer
  - _shadingtable (DWORD) — color translation LUT
