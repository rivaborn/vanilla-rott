# rott/texture.h

## File Purpose
Public interface for texture rendering in the 3D Realms Rise of the Triad engine. Declares global state for texture coordinates, scaling factors, and buffer pointers, plus the core texture rasterization function.

## Core Responsibilities
- Export texture coordinate and dimension state (u, v, count, du, dv)
- Declare source and destination buffer pointers for texture data
- Provide TextureLine() as the main texture rasterization entry point
- Coordinate texture setup before rendering operations

## Key Types / Data Structures
None (header declares only primitive types; data structures defined elsewhere).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| texture_u | int32 | global | Current U texture coordinate |
| texture_v | int32 | global | Current V texture coordinate |
| texture_count | int32 | global | Texture dimension or iteration count |
| texture_du | int32 | global | Delta/step for U coordinate |
| texture_dv | int32 | global | Delta/step for V coordinate |
| texture_source | byte* | global | Source texture buffer pointer |
| texture_dest | byte* | global | Destination framebuffer pointer |
| texture_destincr | byte* | global | Destination increment/stride pointer |

## Key Functions / Methods

### TextureLine
- Signature: `void TextureLine(void)`
- Purpose: Render a single line of textured pixels using the state variables
- Inputs: Reads global texture state (u, v, du, dv, source, dest, destincr)
- Outputs/Return: None; writes to destination buffer
- Side effects: Modifies framebuffer at `texture_dest`; may update texture coordinates
- Calls: Not inferable from this file
- Notes: Typical usage pattern is: set texture_u/v/du/dv and buffer pointers, then call TextureLine()

## Control Flow Notes
This header is part of the rendering pipeline. Callers likely set up global texture state (source coordinates, deltas, buffer pointers) and then call TextureLine() repeatedly per scanline. Indicates a scanline-based texture mapper common in software 3D engines of the early 1990s.

## External Dependencies
- Type definitions: `int32`, `byte` (defined elsewhere, likely in a common types header)
- No visible includes in this file
