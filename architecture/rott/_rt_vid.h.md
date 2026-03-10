# rott/_rt_vid.h

## File Purpose
Private header for RT_VID.C containing video/graphics utility macros. Provides line-drawing and pixel-manipulation abstractions that wrap lower-level VL_* functions, converting from endpoint notation to start-point + length notation.

## Core Responsibilities
- Define pixel-to-block conversion constant (PIXTOBLOCK)
- Provide horizontal line drawing macro wrappers (VW_Hlin, VW_THlin)
- Provide vertical line drawing macro wrappers (VW_Vlin, VW_TVlin)
- Abstract away coordinate calculation logic from call sites

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. This file contains only macro definitions. The macros themselves are simple coordinate converters:

### VW_Hlin (macro)
- Signature: `VW_Hlin(x, z, y, c)`
- Purpose: Draw horizontal line from x to z at y-coordinate with color c
- Converts endpoint notation to VL_Hlin's start + length format
- Expands to: `VL_Hlin(x, y, (z)-(x)+1, c)`

### VW_Vlin (macro)
- Signature: `VW_Vlin(y, z, x, c)`
- Purpose: Draw vertical line from y to z at x-coordinate with color c
- Converts endpoint notation to VL_Vlin's start + length format
- Expands to: `VL_Vlin(x, y, (z)-(y)+1, c)`

### VW_THlin (macro)
- Signature: `VW_THlin(x, z, y, up)`
- Purpose: Draw textured horizontal line from x to z at y-coordinate with update flag up
- Expands to: `VL_THlin(x, y, (z)-(x)+1, up)`

### VW_TVlin (macro)
- Signature: `VW_TVlin(y, z, x, up)`
- Purpose: Draw textured vertical line from y to z at x-coordinate with update flag up
- Expands to: `VL_TVlin(x, y, (z)-(y)+1, up)`

## Control Flow Notes
Not inferable from this file. Used as utility abstractions during rendering pipeline, likely called during frame drawing phases.

## External Dependencies
- Assumes VL_Hlin, VL_Vlin, VL_THlin, VL_TVlin are defined elsewhere (likely in VID-related modules)
- No explicit includes in this file
- Include guard `_rt_vid_private` prevents multiple inclusion
