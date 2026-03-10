# rott/rt_fc_a.h

## File Purpose
Header file declaring low-level assembly functions for floor, ceiling, and sky rendering in the software ray-casting engine. Defines calling conventions and parameter mappings for optimized pixel-drawing routines written in x86 assembly.

## Core Responsibilities
- Declare floor/ceiling row drawing functions
- Declare sky post (vertical column) rendering function
- Declare rotation-based row drawing variants (standard and masked)
- Define Watcom C calling convention mappings via `#pragma aux` directives

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### DrawSkyPost
- Signature: `void DrawSkyPost(byte *buf, byte *src, int height)`
- Purpose: Draw a vertical column of the sky dome
- Inputs: 
  - `buf` (EDI): destination buffer pointer
  - `src` (ESI): source sky texture pointer
  - `height` (ECX): column height in pixels
- Outputs/Return: None (void)
- Side effects: Writes directly to framebuffer; modifies eax, ecx, edx, edi, esi, ebx
- Calls: (Implementation in assembly, not visible here)
- Notes: Raw register-level interface; called per-screen column during sky rendering

### DrawRow
- Signature: `void DrawRow(int count, byte *dest, byte *src)`
- Purpose: Copy/draw a horizontal row of pixels from source to destination
- Inputs:
  - `count` (ECX): number of pixels to copy
  - `dest` (EDI): destination buffer pointer
  - `src` (ESI): source pixel buffer pointer
- Outputs/Return: None (void)
- Side effects: Writes to framebuffer; modifies eax, ebx, ecx, edx, esi, edi
- Calls: (Implementation in assembly, not visible here)
- Notes: Basic pixel copy; likely used for floor/ceiling direct mapping

### DrawRotRow
- Signature: `void DrawRotRow(int count, byte *dest, byte *src)`
- Purpose: Draw a rotated/scaled row of pixels with perspective correction
- Inputs: Same parameter layout as DrawRow
- Outputs/Return: None (void)
- Side effects: Writes to framebuffer; modifies eax, ebx, ecx, edx, esi, edi
- Calls: (Implementation in assembly, not visible here)
- Notes: Performs rotation/perspective sampling during iteration

### DrawMaskedRotRow
- Signature: `void DrawMaskedRotRow(int count, byte *dest, byte *src)`
- Purpose: Draw a rotated row with masking (transparency/palette skipping)
- Inputs: Same parameter layout as DrawRow
- Outputs/Return: None (void)
- Side effects: Writes to framebuffer; modifies eax, ebx, ecx, edx, esi, edi
- Calls: (Implementation in assembly, not visible here)
- Notes: Skips masked pixels (likely color 0 or palette index) during rotation drawing

## Control Flow Notes
These functions are core to the software rendering pipeline's frame-drawing phase. Called per-scanline or per-segment during floor/ceiling and sky rasterization. The rotation variants support perspective-correct texture mapping in the ray-caster's final composition stage.

## External Dependencies
- None visible (pure declarations)
- Implementations defined elsewhere (likely `rt_fc_a.asm`)
- Watcom C `#pragma aux` indicates x86 calling convention for OpenWatcom/Borland C compilers
