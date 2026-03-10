# rott/texture.asm

## File Purpose
Low-level x86-32 assembly implementation of texture-mapped scanline rendering. The `TextureLine_` procedure samples from a source texture bitmap and writes pixels to a destination buffer, advancing through UV coordinates with per-pixel increments for perspective-correct texture mapping.

## Core Responsibilities
- Setup and patch runtime values (du/dv increments, texture/destination addresses) into immediate operands via self-modifying code
- Iterate over pixels in a scanline, computing fixed-point texture coordinates
- Perform texture memory lookup (8-bit paletted texture access)
- Write pixels to destination buffer via per-pixel offset indirection

## Key Types / Data Structures
None.

| Name | Kind | Purpose |
|------|------|---------|
| _texture_u | extern DWORD | Current U coordinate (fixed-point, held in esi) |
| _texture_v | extern DWORD | Current V coordinate (fixed-point, held in ebx) |
| _texture_du | extern DWORD | U increment per pixel (self-modified into patch1) |
| _texture_dv | extern DWORD | V increment per pixel (self-modified into patch2) |
| _texture_count | extern DWORD | Pixel count for this scanline (held in ecx) |
| _texture_source | extern DWORD | Texture bitmap base address (self-modified into patch3) |
| _texture_dest | extern DWORD | Destination buffer base address (self-modified into patch4) |
| _texture_destincr | extern DWORD | Pointer to per-pixel destination offsets (held in edi) |

## Global / File-Static State
None (all state passed via external globals and self-modified immediates).

## Key Functions / Methods

### TextureLine_
- Signature: `void TextureLine_(void)` — no formal parameters; all inputs via external DWORD globals.
- Purpose: Render a single scanline of textured geometry by iterating over pixels, advancing U/V coordinates, sampling texture memory, and writing to framebuffer.
- Inputs:
  - `_texture_u`, `_texture_v`: starting UV coordinates (fixed-point)
  - `_texture_du`, `_texture_dv`: per-pixel UV increments
  - `_texture_count`: number of pixels to render
  - `_texture_source`: base address of source texture bitmap
  - `_texture_dest`: base address of destination framebuffer
  - `_texture_destincr`: pointer to array of per-pixel destination offsets
- Outputs/Return: None (modifies destination buffer in-place).
- Side effects:
  - Self-modifying code: patches four immediate operands (patch1–patch4) with runtime values in the setup phase.
  - Writes 8-bit pixel values directly to destination memory.
  - Reads 8-bit paletted texture values from source.
- Calls: None.
- Notes:
  - Self-modifying code avoids indirect register/memory addressing in the tight inner loop for performance.
  - Fixed-point math: V shifted right 10 bits and implicitly scaled by 64 (texture pitch); U shifted right 16 bits.
  - Inner loop unrolled and aligned at 16 bytes for cache efficiency.
  - `_texture_destincr` treated as an array of DWORD offsets, indexed by `edi`, suggesting non-linear or dynamically-computed framebuffer layout.
  - Loop count (ecx) decrements each iteration; exits when zero.

## Control Flow Notes
- **Setup phase** (~13 instructions): patches immediate operands at labels patch1, patch2, patch3, patch4 with values from external globals.
- **Render loop** (`textureloop`): executes `_texture_count` times. Each iteration:
  1. Advance V and U by fixed-point increments.
  2. Extract integer parts via right-shift.
  3. Compute source texture offset by adding U and V components.
  4. Fetch per-pixel destination offset.
  5. Compute final destination address.
  6. Load source pixel; write to destination.
  7. Decrement count; branch if non-zero.
- Fits into the **rasterization/span-filling** phase of the render pipeline for textured surfaces.

## External Dependencies
- All symbols are `EXTRN` (defined elsewhere, likely in C).
- No system calls or library invocations.
- Direct memory access only (texture read, framebuffer write).
