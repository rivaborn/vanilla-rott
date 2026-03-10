# rott/myprint.h

## File Purpose
Text rendering and output module for the game engine. Provides a color-aware text drawing interface for displaying characters, strings, formatted output, and framed text boxes on-screen.

## Core Responsibilities
- Define standard 16-color palette (BLACK through WHITE)
- Provide low-level character/string output at screen coordinates
- Support formatted printf-style string printing
- Draw text boxes and frames with borders
- Manage text cursor positioning

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| COLORS | enum | 16-color palette: black, blue, green, cyan, red, magenta, brown, light variants, yellow, white |

## Global / File-Static State
None.

## Key Functions / Methods

### DrawText
- Signature: `void DrawText( int x, int y, int ch, int foreground, int background );`
- Purpose: Draw a single character at specified screen coordinates with color
- Inputs: x, y (coordinates), ch (character code), foreground/background (COLORS enum values)
- Outputs/Return: void
- Side effects: Modifies screen buffer
- Calls: Not visible from header

### TextBox
- Signature: `void TextBox( int x1, int y1, int x2, int y2, int ch, int foreground, int background );`
- Purpose: Fill rectangular region with repeating character
- Inputs: x1, y1, x2, y2 (bounds), ch (fill char), foreground/background (colors)
- Outputs/Return: void
- Side effects: Modifies screen buffer

### TextFrame
- Signature: `void TextFrame( int x1, int y1, int x2, int y2, int type, int foreground, int background );`
- Purpose: Draw bordered frame around region
- Inputs: x1, y1, x2, y2 (bounds), type (SINGLE_FRAME=-1 or DOUBLE_FRAME=-2), foreground/background
- Outputs/Return: void
- Side effects: Modifies screen buffer

### myprintf
- Signature: `int myprintf( char *fmt, ... );`
- Purpose: Variadic formatted string printing (printf-style)
- Inputs: fmt (format string), ... (arguments)
- Outputs/Return: int (characters printed)
- Side effects: Writes to screen
- Calls: Likely uses printstring, printnum, printunsigned internally

### printstring, printnum, printunsigned
- Output functions for strings and numbers in various bases
- Return int (character count); operate at current cursor position
- Notes: printunsigned supports arbitrary radix (base) parameter

### mysetxy, myputch
- Position cursor and output single character
- Low-level primitives for direct screen access

## Control Flow Notes
Called during rendering/UI display phases to output text, menus, and debug overlays. No fixed frame timing—functions invoked on-demand.

## External Dependencies
- Underlying screen buffer or console interface (implementation not visible)
- Assumes 16-color DOS-style text mode
