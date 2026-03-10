# rott/_rt_draw.h

## File Purpose
Private header for the rendering subsystem, defining constants, macros, and function declarations for drawing player weapons, transforming 3D geometry, and managing sprite rendering and lighting in the game's frame loop.

## Core Responsibilities
- Define rendering configuration constants (Z-buffer limits, visibility thresholds, height scaling factors)
- Declare weapon drawing and sprite rendering functions
- Declare geometric transformation functions (plane transformation, rotation calculation)
- Define screensaver state structure for menu animation
- Conditionally define weapon graphics count based on build variant (shareware vs. full)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| screensaver_t | struct | Holds animated screensaver state: position (x, y), angle, scale, their deltas, animation phase, time, and pause state |

## Global / File-Static State
None.

## Key Functions / Methods

### DrawPlayerWeapon
- Signature: `void DrawPlayerWeapon(void);`
- Purpose: Render the player's currently equipped weapon to the screen
- Inputs: None (reads global player/weapon state)
- Outputs/Return: None
- Side effects: Modifies framebuffer/screen state
- Calls: Not visible in this file
- Notes: Called during render phase; behavior depends on `W_CHANGE` macro (whether weapon is being raised/lowered)

### TransformPlane
- Signature: `boolean TransformPlane(int x1, int y1, int x2, int y2, visobj_t * plane);`
- Purpose: Transform a 2D plane segment into 3D camera space for rendering
- Inputs: (x1, y1, x2, y2) = plane endpoints; plane = visual object to transform
- Outputs/Return: boolean = success/failure
- Side effects: Modifies plane structure
- Calls: Not visible in this file
- Notes: Likely involves perspective projection and Z-culling using MINZ constant

### CalcRotate
- Signature: `int CalcRotate(objtype *ob);`
- Purpose: Calculate rotation angle for a game object
- Inputs: ob = object to calculate rotation for
- Outputs/Return: int = rotation value (likely in fixed-point or angle units)
- Side effects: None apparent
- Calls: Not visible in this file
- Notes: Used for sprite animation and orientation

### DrawScaleds
- Signature: `void DrawScaleds(void);`
- Purpose: Render all scaled sprites (objects with variable scale/distance)
- Inputs: None (reads global sprite list)
- Outputs/Return: None
- Side effects: Modifies framebuffer
- Calls: Not visible in this file
- Notes: Main entry point for sprite rendering loop

### SetSpriteLightLevel
- Signature: `void SetSpriteLightLevel(int x, int y, visobj_t * sprite, int dir, int fullbright);`
- Purpose: Calculate and set lighting color for a sprite based on position and direction
- Inputs: (x, y) = world position; sprite = sprite object; dir = direction/facing; fullbright = brightness override flag
- Outputs/Return: None
- Side effects: Modifies sprite lighting state
- Calls: Not visible in this file
- Notes: Used for dynamic lighting and shadow/brightness calculation

## Control Flow Notes
This module is part of the **render phase** of the frame loop. `DrawScaleds()` appears to be a main entry point called during screen refresh. `DrawPlayerWeapon()` renders the HUD weapon overlay. Transformation and lighting functions are called internally during sprite/object rendering to handle 3D projection and per-pixel lighting.

## External Dependencies
- **develop.h**: Build configuration flags (SHAREWARE, TEXTMENUS, etc.) controlling conditional compilation
- **Implicit types** (defined elsewhere):
  - `objtype`: game object structure
  - `visobj_t`: visible/renderable object structure
  - `boolean`: boolean type (likely typedef'd)
