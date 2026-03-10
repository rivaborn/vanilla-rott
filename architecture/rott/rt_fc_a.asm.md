# rott/rt_fc_a.asm

## File Purpose

Low-level x86-32 assembly rendering engine for texture-mapped row drawing with rotation, masking, and color translation. Provides four entry points for different pixel-drawing strategies (linear, rotated, masked-rotated, sky columns) optimized via fixed-point arithmetic and self-modifying code patches.

## Core Responsibilities

- **DrawRow_**: Linearly-interpolated texture mapping with color translation
- **DrawRotRow_**: Rotated texture mapping with bounds checking (x: [0–511], y: [0–255])
- **DrawMaskedRotRow_**: Rotated texture mapping with per-pixel masking (skip 0xFF values)
- **DrawSkyPost_**: Vertical sky/background column rendering with shading table lookup
- Code patching: Self-modifying code to inject runtime parameters (xstep, ystep) into instruction immediates
- Pixel-pair loop unrolling for throughput optimization

## Key Types / Data Structures

None.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `loopcount` | `dword` | static | Iteration counter (pixel count / 2) |
| `pixelcount` | `dword` | static | Total pixels to render in this call |
| `_mr_xstep` | `dword` | external | Fractional X-step per pixel (16.16 fixed-point) |
| `_mr_ystep` | `dword` | external | Fractional Y-step per pixel (16.16 fixed-point) |
| `_mr_xfrac` | `dword` | external | Initial X fractional coordinate |
| `_mr_yfrac` | `dword` | external | Initial Y fractional coordinate |
| `_mr_dest` | `dword` | external | Destination framebuffer pointer |
| `_shadingtable` | `dword` | external | Color lookup table (palette/brightness) |

## Key Functions / Methods

### DrawRow_

- **Signature:** `void DrawRow_(ecx=pixel_count, esi=source, edi=dest, ebp=frac)`
- **Purpose:** Draw a horizontally-interpolated textured row with color translation.
- **Inputs:** `ecx` = pixel count; `esi` = texture source pointer; `edi` = framebuffer dest; external `_mr_xstep`, `_mr_ystep`, `_mr_xfrac`, `_mr_yfrac`, `_shadingtable`
- **Outputs/Return:** Pixels written to `[edi]`; registers trashed.
- **Side effects:** Modifies code at `hpatch1`, `hpatch2` (self-modifying); reads/writes framebuffer and static variables.
- **Calls:** None (inline fixed-point arithmetic).
- **Notes:** Processes pixels in pairs. Uses SHLD for fixed-point address calculation (mask 16383 ≈ 14-bit index). Bounds are implicit in texture size. Odd pixel count handled separately.

### DrawRotRow_

- **Signature:** `void DrawRotRow_(ecx=pixel_count, esi=source, edi=dest)`
- **Purpose:** Draw rotated/perspectively-mapped textured row with explicit bounds checking.
- **Inputs:** `ecx` = pixel count; `esi` = texture source; `edi` = dest; external `_mr_xstep`, `_mr_ystep`, `_mr_xfrac`, `_mr_yfrac`
- **Outputs/Return:** Pixels written to `[edi]`.
- **Side effects:** Modifies code at `hrpatch1–hrpatch2c`; reads texture, writes framebuffer.
- **Calls:** None.
- **Notes:** Clamps x to [0–511] and y to [0–255]; out-of-bounds pixels map to offset 0. Pair-loop with early bounds-check branches (`nok3`, `nok4`, `nok1`, `nok2`).

### DrawMaskedRotRow_

- **Signature:** `void DrawMaskedRotRow_(ecx=pixel_count, esi=source, edi=dest)`
- **Purpose:** Rotated texture drawing with per-pixel masking (skip pixels = 0xFF).
- **Inputs:** `ecx` = pixel count; `esi` = texture; `edi` = dest; same external state as `DrawRotRow_`.
- **Outputs/Return:** Selective pixels written; masked pixels left unchanged.
- **Side effects:** Self-modifying code patches (`mhrpatch1–mhrpatch2c`); bounds checking + mask checks.
- **Calls:** None.
- **Notes:** Adds `cmp bl, 0FFh` / `je skip1` guards before writes. Identical bounds logic to `DrawRotRow_`.

### DrawSkyPost_

- **Signature:** `void DrawSkyPost_(ecx=length, esi=source, edi=dest_top)`
- **Purpose:** Draw vertical sky column (post) with interleaved color translation.
- **Inputs:** `ecx` = vertical pixel count; `esi` = column source; `edi` = framebuffer start; `_shadingtable` = color LUT.
- **Outputs/Return:** Pixels written at `[edi]` and `[edi – SCREENROW]` (interleaved vertically).
- **Side effects:** Advances `esi` by 2, `edi` by `SCREENROW*2` per iteration.
- **Calls:** None.
- **Notes:** `SCREENROW=96` (pixel stride per screen line). Processes 16-bit pairs (two vertical pixels per iteration). Odd-length columns handled in `dsextra`.

## Control Flow Notes

**Initialization:** Each function unpacks `_mr_xstep`, `_mr_ystep`, `_mr_xfrac`, `_mr_yfrac` and writes them into instruction immediates via self-modifying code (patches at `hpatch1`, etc.). This amortizes parameter passing.

**Main loop (Rows):** Pair-loop unrolls two pixels per iteration, using SHLD + AND for fixed-point coordinate→texture-index. Bounds checking (rotation functions) branches on out-of-bounds; in-bounds branch falls through to pixel fetch.

**Final pixel:** After loop exits (pair-count=0), final odd pixel handled if `pixelcount & 1 = 1`.

**Shutdown:** Functions clean stack and return.

## External Dependencies

- **Includes:** `<indirect>` — Turbo Assembler (TASM) directives (`.386P`, `.MODEL`, `IDEAL` block syntax).
- **External symbols:** `_mr_xstep`, `_mr_ystep`, `_mr_xfrac`, `_mr_yfrac`, `_mr_rowofs`, `_mr_count`, `_mr_dest`, `_shadingtable` — defined elsewhere, presumed set by C caller before invocation.
- **Implicit:** Framebuffer layout (destination array layout), texture format (byte-indexed pixels), and shading table format (256-entry byte LUT) inferred from usage.
