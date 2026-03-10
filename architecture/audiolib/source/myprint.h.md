# audiolib/source/myprint.h

## File Purpose
This header declares text rendering and formatting primitives for screen output. It provides colored text drawing, cursor positioning, and printf-like output functions, suggesting it's a display utility module used across the engine despite being in the audiolib directory.

## Core Responsibilities
- Define standard 16-color palette (DOS/early graphics colors)
- Declare character-level drawing primitives (DrawText, TextBox, TextFrame)
- Provide cursor-based text output functions (myputch, printstring)
- Declare formatted printing with variable arguments (myprintf)
- Support arbitrary radix integer conversion (printunsigned)
- Define frame decoration constants (SINGLE_FRAME, DOUBLE_FRAME)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| COLORS | enum | 16-color palette: BLACK, BLUE, GREEN, CYAN, RED, MAGENTA, BROWN, LIGHTGRAY, DARKGRAY, LIGHTBLUE, LIGHTGREEN, LIGHTCYAN, LIGHTRED, LIGHTMAGENTA, YELLOW, WHITE |

## Global / File-Static State
None.

## Key Functions / Methods

### DrawText
- Signature: `void DrawText( int x, int y, int ch, int foreground, int background );`
- Purpose: Draw a single character at screen coordinates with colors
- Inputs: x, y (screen position); ch (character code); foreground, background (COLORS enum values)
- Outputs/Return: void
- Side effects: Writes to screen buffer
- Calls: Not visible in header
- Notes: Direct primitive for character placement; foreground/background likely map to COLORS enum

### TextBox
- Signature: `void TextBox( int x1, int y1, int x2, int y2, int ch, int foreground, int background );`
- Purpose: Fill a rectangular region with a character and colors
- Inputs: x1, y1, x2, y2 (bounding box); ch (fill character); colors
- Outputs/Return: void
- Side effects: Writes to screen buffer
- Calls: Not visible in header

### TextFrame
- Signature: `void TextFrame( int x1, int y1, int x2, int y2, int type, int foreground, int background );`
- Purpose: Draw a rectangular frame border (single or double-line)
- Inputs: x1, y1, x2, y2 (bounding box); type (SINGLE_FRAME=-1 or DOUBLE_FRAME=-2); colors
- Outputs/Return: void
- Side effects: Writes frame characters to screen buffer
- Calls: Not visible in header

### mysetxy, myputch, printstring, printnum
- Manage cursor-based text output: set position, print character, print strings/integers

### printunsigned
- Signature: `int printunsigned( unsigned long number, int radix );`
- Purpose: Output unsigned integer in specified base (2–36 expected)
- Inputs: number, radix (base for conversion)
- Outputs/Return: int (likely characters printed)
- Side effects: Advances cursor, writes to screen
- Calls: Not visible in header

### myprintf
- Signature: `void myprintf( char *fmt, ... );`
- Purpose: Printf-like formatted output at current cursor
- Inputs: format string, variable arguments
- Outputs/Return: void
- Side effects: Advances cursor, writes formatted text to screen
- Calls: Not visible in header

## Control Flow Notes
Utility library called on-demand by higher-level code. Cursor state (set by mysetxy) affects subsequent output. No initialization or frame-loop integration evident.

## External Dependencies
None (header-only declarations). No includes. Uses COLORS enum values and frame constants defined locally.
