# rott/rt_floor.h

## File Purpose
Public header declaring the interface for floor, ceiling, and sky rendering functionality in the ray-casting renderer. Exposes ray-marching state variables and plane-drawing entry points needed by the rendering pipeline.

## Core Responsibilities
- Declare floor and ceiling plane rendering functions
- Expose ray-marching step/fraction state for coordinate calculations
- Provide sky rendering API (generation, drawing, visibility checks)
- Manage parallax sky enable/disable flag

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `mr_xstep` | int | extern | Ray-march X step increment for plane rendering |
| `mr_ystep` | int | extern | Ray-march Y step increment for plane rendering |
| `mr_xfrac` | int | extern | Ray-march X fractional component |
| `mr_yfrac` | int | extern | Ray-march Y fractional component |
| `sky` | int | extern | Boolean flag: parallax sky enabled (1) or disabled (0) |

## Key Functions / Methods

### DrawPlanes
- **Signature:** `void DrawPlanes(void)`
- **Purpose:** Render floor and ceiling planes for the current frame.
- **Inputs:** None (uses global state: `mr_xstep`, `mr_ystep`, `mr_xfrac`, `mr_yfrac`)
- **Outputs/Return:** None
- **Side effects:** Modifies video memory / framebuffer
- **Calls:** Implementation in rt_floor.c
- **Notes:** Likely called once per frame during main render pass.

### SetPlaneViewSize
- **Signature:** `void SetPlaneViewSize(void)`
- **Purpose:** Configure viewport dimensions or scaling for plane rendering.
- **Inputs:** None (reads from video mode / engine state)
- **Outputs/Return:** None
- **Side effects:** Updates internal plane-rendering state
- **Calls:** Implementation in rt_floor.c
- **Notes:** Likely called on resolution change or engine init.

### MakeSkyTile
- **Signature:** `void MakeSkyTile(byte *tile)`
- **Purpose:** Generate or populate a single sky texture tile.
- **Inputs:** `tile` — pointer to byte buffer for sky tile data
- **Outputs/Return:** Tile data written to input buffer
- **Side effects:** Modifies tile buffer
- **Calls:** Implementation in rt_floor.c
- **Notes:** Used to construct sky textures dynamically.

### DrawFullSky
- **Signature:** `void DrawFullSky(void)`
- **Purpose:** Render the complete sky background.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies video memory
- **Calls:** Implementation in rt_floor.c
- **Notes:** Likely checks `sky` flag before drawing.

### SkyExists
- **Signature:** `boolean SkyExists(void)`
- **Purpose:** Query whether sky is available / enabled.
- **Inputs:** None
- **Outputs/Return:** `boolean` — true if parallax sky is active
- **Side effects:** None
- **Calls:** Implementation in rt_floor.c
- **Notes:** Convenience check, likely equivalent to `(sky != 0)`.

## Control Flow Notes
These functions are part of the **render phase** of the frame loop:
- `SetPlaneViewSize()` called during **engine init** or **viewport resize**.
- `DrawPlanes()` and `DrawFullSky()` called during **per-frame rendering** (likely after screen clear).
- Ray-march state (`mr_xstep`, etc.) updated by the ray-casting core before each plane draw.

## External Dependencies
- **Includes/Imports:** None visible (this is a public header; implementation in rt_floor.c).
- **External symbols:** Basic C types only (`int`, `byte`, `boolean`, `void`); implementations defined elsewhere.
