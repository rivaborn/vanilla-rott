# rott/f_scale.h

## File Purpose
Header file declaring the public interface for film column scaling and rendering operations. Exports global scaling parameters and two key rendering functions used by the engine's vertical scaling subsystem, likely for drawing stretched/scaled sprite columns.

## Core Responsibilities
- Export global scaling state variables (Y-bounds, scale factor, texture mid-point)
- Declare the core R_DrawFilmColumn function for rendering scaled columns
- Declare the DrawFilmPost function for post-processing film output
- Maintain consistent parameter passing conventions via pragma directives

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| cin_yl | int | extern | Lower Y bound for column clipping |
| cin_yh | int | extern | Upper Y bound for column clipping |
| cin_ycenter | int | extern | Y center coordinate for scaling reference |
| cin_iscale | int | extern | Inverse/scale factor for column height mapping |
| cin_texturemid | int | extern | Vertical texture center for alignment |
| cin_source | byte * | extern | Source buffer pointer for scaled data |

## Key Functions / Methods

### R_DrawFilmColumn
- Signature: `void R_DrawFilmColumn(byte * buf)`
- Purpose: Render a scaled column of pixel data to the output buffer
- Inputs: `buf` — destination buffer pointer (passed in EDI register)
- Outputs/Return: None (void)
- Side effects: Writes to output buffer; uses global scaling state (cin_yl, cin_yh, cin_iscale, etc.)
- Calls: Not inferable from this file
- Notes: Watcom C pragma specifies exact register usage and clobbering; performance-critical function

### DrawFilmPost
- Signature: `void DrawFilmPost(byte * buf, byte * src, int height)`
- Purpose: Post-process or copy scaled film data with specified height
- Inputs: `buf` (EDI) — destination buffer, `src` (ESI) — source buffer, `height` (ECX) — column height in pixels
- Outputs/Return: None (void)
- Side effects: Writes to output buffer; modifies registers
- Calls: Not inferable from this file
- Notes: Watcom C pragma specifies register-optimized calling convention

## Control Flow Notes
Part of the rendering subsystem; called during frame rendering to draw scaled vertical strips (columns) of sprites or textures. The global cin_* variables are populated upstream and used by these functions.

## External Dependencies
- Standard C types (byte, int, void)
- Watcom C compiler pragmas (`#pragma aux`) for register allocation and calling conventions
- Assumed to be linked with implementation files that use these declarations
