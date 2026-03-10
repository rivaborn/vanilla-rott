# rott/rt_build.h

## File Purpose
Public header declaring the menu buffer management interface for the ROTT engine. Provides functions to initialize, render, and draw UI elements (shapes, text, pictures) to an off-screen menu buffer that can be positioned and displayed with transparency/intensity effects.

## Core Responsibilities
- Menu buffer lifecycle management (setup, shutdown, clear)
- Menu buffer positioning and animation (angle, distance, refresh timing)
- Drawing primitives to the menu buffer (shapes, pictures, text in multiple styles)
- Text rendering variants (proportional, transparent, colored, shaded)
- Menu UI element erasure and region management

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Menuflipspeed` | `int` | extern | Controls animation speed for menu display transitions |
| `intensitytable` | `byte *` | extern | Lookup table for intensity/transparency effects on menu buffer |

## Key Functions / Methods

### SetupMenuBuf / ShutdownMenuBuf
- Signature: `void SetupMenuBuf(void)` / `void ShutdownMenuBuf(void)`
- Purpose: Initialize and clean up menu buffer resources
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates/deallocates menu buffer memory

### PositionMenuBuf
- Signature: `void PositionMenuBuf(int angle, int distance, boolean drawbackground)`
- Purpose: Position the menu buffer in screen space with optional background
- Inputs: Angle and distance (polar coords), background flag
- Outputs/Return: None
- Side effects: Updates menu buffer transform state

### RefreshMenuBuf / FlipMenuBuf
- Signature: `void RefreshMenuBuf(int time)` / `void FlipMenuBuf(void)`
- Purpose: Update animation state and display menu buffer to screen
- Inputs: `time` for animation timing
- Outputs/Return: None
- Side effects: Modifies visual display

### Draw Functions (Multiple Variants)
- **Variants**: `DrawMenuBufItem`, `DrawTMenuBufItem`, `DrawIMenuBufItem`, `DrawColoredMenuBufItem`, `DrawMenuBufPic`, `DrawTMenuBufPic`
- **Signature**: `void Draw*(int x, int y, int shapenum [, int color])`
- **Purpose**: Render shapes/pictures to menu buffer with different blending modes (opaque, transparent, intensity-based, colored)
- **Inputs**: Screen coordinates, shape ID, optional color value
- **Outputs/Return**: None
- **Side effects**: Modifies menu buffer pixel data

### Text Rendering Functions
- **Functions**: `DrawMenuBufIString`, `DrawMenuBufPropString`, `DrawTMenuBufPropString`, `MenuBufCPrint*`, `MenuBufPrint`
- **Purpose**: Render strings in various styles (proportional, shaded, line-wrapped)
- **Inputs**: Position (px, py), text string, optional shade/color
- **Outputs/Return**: None
- **Side effects**: Modifies menu buffer with text

### Box / Line Primitives
- **Functions**: `DrawTMenuBufBox`, `DrawTMenuBufHLine`, `DrawTMenuBufVLine`
- **Purpose**: Draw geometric primitives (transparent boxes, horizontal/vertical lines)
- **Inputs**: Position, dimensions, optional "up" lighting direction
- **Outputs/Return**: None

### EraseMenuBufRegion / SetAlternateMenuBuf
- **Signature**: `void EraseMenuBufRegion(int x, int y, int width, int height)` / `void SetAlternateMenuBuf(void)`
- **Purpose**: Clear rectangular regions; switch to alternate buffer
- **Inputs**: Region coordinates and dimensions
- **Outputs/Return**: None
- **Side effects**: Clears buffer memory or swaps active buffer

## Control Flow Notes
Part of the menu/UI rendering pipeline. Typical frame flow:
1. **Init**: `SetupMenuBuf()` at engine startup
2. **Per-Frame**: `ClearMenuBuf()` → series of `Draw*()` calls → `RefreshMenuBuf()` → `FlipMenuBuf()`
3. **Shutdown**: `ShutdownMenuBuf()` on exit

The menu buffer appears to be a double-buffered off-screen surface with support for transparency, intensity mapping, and flexible positioning.

## External Dependencies
- `byte` type (likely defined in a common types header; maps to `unsigned char`)
- `boolean` type (likely `typedef` for `int` or similar)
- Implementation in `rt_build.c`
