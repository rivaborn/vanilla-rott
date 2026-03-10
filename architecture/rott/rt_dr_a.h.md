# rott/rt_dr_a.h

## File Purpose
Header file declaring low-level drawing functions for the software renderer. Exposes optimized assembly routines (`RT_DR_A.ASM`) for column-based rasterization, which form the core pixel-writing stage of the rendering pipeline.

## Core Responsibilities
- Declare graphics mode setup (`SetMode240`)
- Define column drawing primitives for wall rendering (`DrawPost`, `DrawHeightPost`)
- Expose specialized drawing functions for walls, menus, and maps
- Manage refresh buffer clearing
- Define parameter passing conventions and register usage for assembly routines

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### SetMode240
- Signature: `void SetMode240(void)`
- Purpose: Configure graphics mode to 240-line resolution
- Inputs: None
- Outputs/Return: None
- Side effects: Sets video mode globally
- Calls: Not visible (likely in assembly)
- Notes: Inferred to be initialization/mode-setting

### RefreshClear
- Signature: `void RefreshClear(void)`
- Purpose: Clear the refresh/frame buffer for next frame
- Inputs: None
- Outputs/Return: None
- Side effects: Clears global frame buffer
- Calls: Not visible (likely in assembly)

### DrawPost
- Signature: `void DrawPost(int height, char * column, byte * buf)`
- Purpose: Write a column of pixels to frame buffer at specified height
- Inputs: `height` (column height), `column` (source pixel data), `buf` (destination buffer)
- Outputs/Return: None
- Side effects: Modifies frame buffer memory
- Calls: Not visible (assembly routine)
- Notes: `#pragma aux` specifies: ECX=height, ESI=column, EDI=buf; modifies eax,ebx,ecx,edx,esi,edi

### DrawHeightPost
- Signature: `void DrawHeightPost(int height, byte * src, byte * buf)`
- Purpose: Draw heightmap-indexed column to buffer
- Inputs: `height`, `src` (heightmap/source data), `buf` (destination)
- Outputs/Return: None
- Side effects: Modifies frame buffer
- Notes: ECX=height, ESI=src, EDI=buf; located in `rt_dr_a.asm`

### R_DrawWallColumn
- Signature: `void R_DrawWallColumn(byte * buf)`
- Purpose: Render a single wall column using internal state
- Inputs: `buf` (destination buffer)
- Outputs/Return: None
- Side effects: Writes to frame buffer; likely uses global wall texture/height state
- Notes: EDI=buf; expects height/texture already configured

### DrawMenuPost / DrawMapPost
- Signature: `void DrawMenuPost(int height, byte * src, byte * buf)` and similar for `DrawMapPost`
- Purpose: Specialized column drawing for UI and map rendering
- Inputs: `height`, `src` (source data), `buf` (destination)
- Outputs/Return: None
- Side effects: Modifies frame buffer
- Notes: ECX=height, ESI=src, EDI=buf; parameter conventions identical to `DrawHeightPost`

## Control Flow Notes
This module sits at the bottom of the rendering pipeline, called during the per-frame rasterization loop. `SetMode240` runs at initialization; `RefreshClear` at frame start; column drawing functions execute during wall/sprite/UI rasterization. These are performance-critical assembly routines, hence the explicit register bindings via `#pragma aux`.

## External Dependencies
- All functions defined elsewhere: `rt_dr_a.asm` (assembly implementations)
- No other includes or external symbols visible in this header
