# rott/r_scale.asm

## File Purpose
Low-level x86 assembly implementation of scaled texture column rendering. Draws vertical slices of textures with interpolation/filtering, operating on fixed-point texture coordinates. Part of the software raycasting/scanline renderer pipeline.

## Core Responsibilities
- Compute texture coordinate interpolation using fixed-point arithmetic
- Fetch and write texture pixels with vertical scaling/filtering
- Process columns in pairs for cache and loop efficiency
- Handle variable column heights and vertical positioning within screen buffer
- Self-modify scale increment instructions for runtime configuration

## Key Types / Data Structures
None (pure procedural assembly; types implicit in register layout).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| loopcount | DWORD | static | Iteration counter for pixel-pair processing loop |
| pixelcount | DWORD | static | Total column height in pixels |
| _dc_yl | DWORD | extern | Top Y coordinate of column to draw |
| _dc_yh | DWORD | extern | Bottom Y coordinate of column |
| _dc_ycenter | DWORD | extern | Center Y (for perspective center) |
| _dc_iscale | DWORD | extern | Inverse scale / texture step size (fixed-point) |
| _dc_texturemid | DWORD | extern | Texture V-coordinate offset |
| _dc_source | DWORD | extern | Pointer to texture data row |
| _ylookup | DWORD | extern | Lookup table mapping Y to screen buffer address |

## Key Functions / Methods

### R_DrawFilmColumn_
- **Signature:** `void R_DrawFilmColumn_(void)` (no args; parameters via globals)
- **Purpose:** Draw a single vertical column of scaled/filtered texture to screen buffer.
- **Inputs:** 
  - Global state: `_dc_yl`, `_dc_yh`, `_dc_ycenter`, `_dc_iscale`, `_dc_texturemid`, `_dc_source`, `_ylookup`
  - `edi` (caller must pre-load with screen buffer base)
- **Outputs/Return:** Pixels written to video memory at address `edi + Y*SCREENWIDTH`
- **Side effects:** 
  - Modifies global static `loopcount`, `pixelcount`
  - **Self-modifying code:** patches `_dc_iscale` value into two `add ebp, 12345678h` instructions (patch1, patch2)
  - Writes to screen memory
- **Calls (direct):** None (inline fixed-point iteration)
- **Notes:**
  - Uses fixed-point arithmetic: `ebp` accumulates texture V-coordinate; upper 16 bits (`shr ecx, 16`) index texture row.
  - Processes pixels in pairs (doubleloop) for efficiency; handles odd final pixel separately (checklast).
  - Early exit if `pixelcount <= 0` (js done).
  - Self-modifying pattern allows dynamic per-column scale without per-pixel multiply; highly optimized for 1990s CPU pipeline.
  - Assumes `SCREENWIDTH = 96` pixels/scanline.

## Control Flow Notes
Initialization phase:
1. Load `_dc_yl` → `ebp`, compute screen address via `_ylookup[ebp*4] + edi`.
2. Calculate pixel count (`_dc_yh - _dc_yl + 1`).
3. Initialize texture fraction: `ebp = _dc_texturemid - (_dc_ycenter - _dc_yl) × _dc_iscale`.
4. Patch `_dc_iscale` into two `add` instructions.

Main loop (doubleloop):
- Iterates `loopcount = pixelcount / 2` times, writing two pixels per iteration.
- Each iteration: shift fraction to integer index, fetch texture pixel, write to screen, advance fraction.
- Screen write addresses alternate: `edi`, `edi + SCREENWIDTH` (two scanlines).

Final check (checklast):
- If `pixelcount` is odd, write final pixel at `edi`.

## External Dependencies
- **Includes/Directives:** None (pure assembly).
- **External symbols:** `_dc_yl`, `_dc_yh`, `_dc_ycenter`, `_dc_iscale`, `_dc_texturemid`, `_ylookup`, `_dc_source` (defined elsewhere; likely C/C++ globals set by rendering state machine).
- **Assumptions:** Caller sets `edi` to screen buffer base; linear video memory layout.
