# rott/rt_str.h

## File Purpose
Public header for the RT_STR.C string/text module. Declares functions for string measurement and rendering (including proportional and intensity-based variants), text input handling, window management, and geometric data structures used throughout the game engine's UI and text output systems.

## Core Responsibilities
- String measurement (proportional, intensity, basic) and rendering (clipped, centered, buffered)
- Numeric printing (signed/unsigned integers with custom radix support)
- Text input handling with line editing, defaults, and constraints
- Window drawing and positioning on screen
- Intensity-based font color mapping and rendering
- Definition of geometric primitives (Point, Rect) and screen state (WindowRec)
- Callback registration for custom measure/print routines

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Point | struct | 2D point with x, y coordinates |
| WindowRec | struct | Screen window state: position (x, y), dimensions (w, h), and saved position (px, py) |
| Rect | struct | Rectangle defined by upper-left and lower-right Points |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| fontcolor | int | global (extern) | Current font color used for text rendering |

## Key Functions / Methods

### VW_DrawClippedString
- Signature: `void VW_DrawClippedString (int x, int y, char *string)`
- Purpose: Draw a string at screen position, clipping to screen bounds
- Inputs: x, y screen coordinates; string pointer
- Outputs/Return: void
- Side effects: Modifies framebuffer; respects fontcolor global
- Calls: (defined elsewhere)
- Notes: Viewport-aware rendering

### VW_DrawPropString / VWB_DrawPropString
- Signature: `void VW_DrawPropString (char *string)` and `void VWB_DrawPropString (char *string)`
- Purpose: Draw a proportional-width string (VWB likely buffered variant)
- Inputs: string pointer
- Outputs/Return: void
- Side effects: Modifies framebuffer; uses fontcolor global
- Calls: (defined elsewhere)
- Notes: Proportional fonts have variable character widths

### VW_MeasurePropString
- Signature: `void VW_MeasurePropString (char *string, int *width, int *height)`
- Purpose: Query dimensions of a proportional string without rendering
- Inputs: string pointer; output pointers
- Outputs/Return: void (dimensions written to *width, *height)
- Side effects: None
- Calls: (defined elsewhere)
- Notes: Used for layout calculations

### US_MeasureStr
- Signature: `void US_MeasureStr (int *width, int *height, char * s, ...)`
- Purpose: Measure string dimensions (variadic interface)
- Inputs: output pointers; format string and arguments
- Outputs/Return: void
- Side effects: None
- Calls: (defined elsewhere)
- Notes: Variadic interface suggests printf-like formatting

### US_SetPrintRoutines
- Signature: `void US_SetPrintRoutines (void (*measure)(char *, int *, int *, font_t *), void (*print)(char *))`
- Purpose: Register custom measure and print function callbacks
- Inputs: Function pointers for custom measure and print
- Outputs/Return: void
- Side effects: Replaces default print/measure implementations globally
- Calls: (defined elsewhere)
- Notes: Strategy pattern for pluggable rendering backends

### US_LineInput
- Signature: `boolean US_LineInput (int x, int y, char *buf, char *def, boolean escok, int maxchars, int maxwidth, int color)`
- Purpose: Interactive line input at screen position with validation and constraints
- Inputs: x, y position; buffer; default string; escape key enabled flag; max chars and pixel width; text color
- Outputs/Return: boolean (ESC cancel vs. ENTER confirm)
- Side effects: Modifies buffer; reads input state
- Calls: (defined elsewhere)
- Notes: Main UI input routine; maxwidth constrains visual width on screen

### US_DrawWindow / US_CenterWindow
- Signature: `void US_DrawWindow (int x, int y, int w, int h)` and `void US_CenterWindow (int w, int h)`
- Purpose: Draw a window frame at position or centered on screen
- Inputs: Coordinates and dimensions
- Outputs/Return: void
- Side effects: Modifies framebuffer; draws border/background
- Calls: (defined elsewhere)
- Notes: Dialog box infrastructure

### DrawIntensityString
- Signature: `void DrawIntensityString (unsigned short int x, unsigned short int y, char *string, int color)`
- Purpose: Draw a string using intensity-based (shaded/gradient) font rendering
- Inputs: x, y position; string; color index
- Outputs/Return: void
- Side effects: Modifies framebuffer
- Calls: (defined elsewhere)
- Notes: Special effect rendering, likely for UI emphasis

### GetIntensityColor
- Signature: `byte GetIntensityColor (byte pix)`
- Purpose: Map a pixel value to an intensity color (palette lookup)
- Inputs: Pixel value
- Outputs/Return: Intensity-mapped byte
- Side effects: None
- Calls: (defined elsewhere)
- Notes: Palette-based color transformation

## Control Flow Notes
This module is a mid-level UI/text rendering abstraction. Inferred usage:
- **Render cycle**: String and window drawing occur during frame rendering for HUD, menus, dialogs
- **Input cycle**: `US_LineInput` and `CalibrateJoystick` called during menu/dialog update phases
- **Initialization**: `US_SetPrintRoutines` likely called during engine startup to select rendering backend
- **Global state**: `fontcolor` is set before rendering calls and persists across the frame

## External Dependencies
- **lumpy.h**: Defines graphics structures (font_t, pic_t, patch_t, etc.) used for asset data
- **myprint.h**: Lower-level text rendering (DrawText, TextBox, TextFrame, myprintf) and color constants
- Defined elsewhere: Video/graphics rendering backend (VWB/VW functions), input system, palette management
