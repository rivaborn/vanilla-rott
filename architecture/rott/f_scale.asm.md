# rott/f_scale.asm

## File Purpose

Optimized x86 assembly implementation for column-based texture scaling and rendering in a software 3D engine. Provides high-performance vertical pixel-column drawing with fixed-point texture filtering, using self-modifying code for dynamic per-call scaling parameters.

## Core Responsibilities

- Draw scaled texture columns to a framebuffer using fixed-point texture coordinate interpolation
- Dynamically configure scaling factors via runtime code patching
- Process pixels in paired batches to maximize throughput
- Provide post-processing layout for interleaved texture data

## Key Types / Data Structures

None.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| loopcount | DWORD | static | Loop iteration counter for paired-pixel processing |
| pixelcount | DWORD | static | Total pixels to draw in current column |
| _cin_yl | DWORD | global | Y-coordinate start (top of column) |
| _cin_yh | DWORD | global | Y-coordinate end (bottom of column) |
| _cin_ycenter | DWORD | global | Y-coordinate reference point for scale calculation |
| _cin_iscale | DWORD | global | Inverse scale factor (fixed-point, 16-bit fractional) |
| _cin_texturemid | DWORD | global | Texture coordinate offset |
| _cin_source | DWORD | global | Pointer to source texture data |

## Key Functions / Methods

### R_DrawFilmColumn_

- **Signature:** `void R_DrawFilmColumn_()`
- **Purpose:** Draw a single vertical column of pixels from a texture to the framebuffer with per-pixel scale interpolation.
- **Inputs (via global state):**
  - `_cin_yl`: Y start (screen pixel row)
  - `_cin_yh`: Y end (screen pixel row)
  - `_cin_ycenter`: Y center reference
  - `_cin_iscale`: Inverse scaling factor (fixed-point: high 16 bits = integer, low 16 bits = fraction)
  - `_cin_texturemid`: Texture coordinate baseline
  - `_cin_source`: Texture data pointer
  - `edi` (EDI register): Screen destination buffer pointer
- **Outputs/Return:** None; modifies framebuffer in-place
- **Side effects:**
  - Modifies framebuffer memory at `edi` and adjacent rows
  - Self-modifying code: patches `patch1` and `patch2` with `_cin_iscale` value
  - Updates `pixelcount` and `loopcount`
- **Calls:** Indirect call to `_ylookup` table (macro-like lookup)
- **Notes:**
  - Uses 16-bit fixed-point texture coordinates (high word = integer index, low word = subpixel fraction)
  - Processes pixels in pairs (two iterations per loop) for throughput
  - Maintains running fractional texture position in `ebp` across loop iterations
  - Self-modifying code pattern allows dynamic scaling per invocation without branching
  - Unrolled loop calculates 4 pixels per iteration (alternating pairs)
  - Handles odd final pixel after main loop
  - Early exit if pixel count ≤ 0

### DrawFilmPost_

- **Signature:** `void DrawFilmPost_()`
- **Purpose:** Rearrange or interleave texture data layout (post-processing).
- **Inputs (via registers):**
  - `edi`: Destination pointer
  - `esi`: Source data pointer
  - `ecx`: Length (number of elements to process)
- **Outputs/Return:** None; modifies destination buffer
- **Side effects:**
  - Writes interleaved/spread data to destination buffer
  - Updates `esi` and `edi` pointers as it processes
- **Calls:** None
- **Notes:**
  - Processes input in pairs; each pair is written to two vertically-offset rows (separated by `SCREENROW*2`)
  - Odd final element handled separately
  - Optimized for cache locality by vertical spreading

## Control Flow Notes

- **Initialization phase:** `R_DrawFilmColumn_` calculates texture coordinate baseline and patches scaling loop with dynamic per-call parameters.
- **Render phase:** Main `doubleloop` processes pixels in a tight unrolled loop, issuing memory reads ahead of writes to hide latency.
- **Finalization:** Handles odd pixel count edge case; cleans up and returns.
- Fits into a per-column render pass, likely called once per visible vertical wall strip in a software raycaster.

## External Dependencies

- **_ylookup** (DWORD array): Lookup table mapping Y screen coordinates to framebuffer byte offsets; indexed by `[_cin_yl * 4]`
- Constants: `SCREENWIDTH = 96`, `SCREENROW = 96`
