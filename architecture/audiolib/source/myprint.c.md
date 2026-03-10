# audiolib/source/myprint.c

## File Purpose

Implements low-level text output and formatting for DOS-era VGA text mode (80×24 characters). Provides direct video memory manipulation (0xb0000), text positioning, frame drawing, and printf-like formatted output with automatic scrolling.

## Core Responsibilities

- Direct character rendering to VGA text memory with color attributes
- Screen cursor positioning and line-based text output
- Rectangular text fill and frame drawing with border characters
- Printf-like format string processing (%d, %s, %u, %x specifiers)
- Automatic screen scrolling when output exceeds display bounds
- Newline/carriage return handling with line clearing

## Key Types / Data Structures

None (color enum and frame type constants defined in header only).

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `disp_offset` | `unsigned short` | static | Current cursor position in video memory (byte offset from 0xb0000); initialized to 160×24 (end of 80×24 screen). |

## Key Functions / Methods

### DrawText
- **Signature:** `void DrawText(int x, int y, int ch, int foreground, int background)`
- **Purpose:** Render a single character at screen position (x, y) with foreground/background color attributes.
- **Inputs:** x, y (0–79, 0–23); ch (ASCII code or NONE=-1 to skip char write); foreground, background (0–15 color indices).
- **Outputs/Return:** None (void).
- **Side effects:** Direct write to video memory at `0xb0000 + y*160 + x*2` and `+x*2+1` (character and attribute bytes).
- **Calls:** None.
- **Notes:** If ch is NONE, only the color attribute is written. Each screen cell requires 2 bytes (char, attr).

### TextBox
- **Signature:** `void TextBox(int x1, int y1, int x2, int y2, int ch, int foreground, int background)`
- **Purpose:** Fill a rectangular region with a uniform character and colors.
- **Inputs:** Rectangle corners (x1, y1) to (x2, y2), fill character ch, colors.
- **Outputs/Return:** None.
- **Side effects:** Fills all cells in rectangle via DrawText calls.
- **Calls:** DrawText (multiple times in nested loop).
- **Notes:** Inefficient for large areas. Boundary condition is inclusive (≤).

### TextFrame
- **Signature:** `void TextFrame(int x1, int y1, int x2, int y2, int type, int foreground, int background)`
- **Purpose:** Draw a rectangular frame border with corners and edges (three styles supported).
- **Inputs:** Rectangle corners, type (0, SINGLE_FRAME=-1, DOUBLE_FRAME=-2), colors.
- **Outputs/Return:** None.
- **Side effects:** Draws frame edges in video memory.
- **Calls:** DrawText (multiple calls per frame edge).
- **Notes:** type=0 draws hollow box; SINGLE_FRAME and DOUBLE_FRAME use extended ASCII line-drawing characters. Three separate conditional blocks.

### mysetxy
- **Signature:** `void mysetxy(int x, int y)`
- **Purpose:** Set the cursor position for subsequent character output.
- **Inputs:** x, y screen coordinates.
- **Outputs/Return:** None.
- **Side effects:** Updates global `disp_offset` to `(x*2) + (y*160)`.
- **Calls:** None.
- **Notes:** Converts 2D coordinates to linear byte offset in video memory.

### myputch
- **Signature:** `void myputch(char ch)`
- **Purpose:** Output a single character with cursor advance, newline/carriage return handling, and automatic scrolling.
- **Inputs:** ch (character code).
- **Outputs/Return:** None.
- **Side effects:** 
  - Writes character to video memory at current `disp_offset`.
  - Updates `disp_offset` after character write and for line breaks.
  - Scrolls screen up by 1 line (moves 160 bytes backward, clears new bottom line) when offset ≥ 160×24.
  - Clears current line on '\n'.
- **Calls:** None (direct memory access).
- **Notes:** Handles printable ASCII (ch ≥ 32), '\r' (line start), '\n' (next line + clear). Scrolling loop moves screen contents backward in memory.

### printstring
- **Signature:** `int printstring(char *string)`
- **Purpose:** Output a null-terminated string by calling myputch for each character.
- **Inputs:** Pointer to string.
- **Outputs/Return:** Count of characters output.
- **Side effects:** Calls myputch for each character.
- **Calls:** myputch.
- **Notes:** Simple linear scan until '\0'.

### printnum
- **Signature:** `int printnum(int number)`
- **Purpose:** Convert signed integer to decimal string and output.
- **Inputs:** number (int).
- **Outputs/Return:** Character count output.
- **Side effects:** Converts via itoa, then prints via printstring.
- **Calls:** itoa (stdlib, not defined here), printstring.
- **Notes:** Uses 100-byte local buffer.

### printunsigned
- **Signature:** `int printunsigned(unsigned long number, int radix)`
- **Purpose:** Convert unsigned integer to string in arbitrary base (10, 16, etc.) and output.
- **Inputs:** number (unsigned long), radix (base).
- **Outputs/Return:** Character count output.
- **Side effects:** Converts via ultoa, then prints via printstring.
- **Calls:** ultoa (stdlib, not defined here), printstring.
- **Notes:** Uses 100-byte local buffer.

### myprintf
- **Signature:** `int myprintf(char *fmt, ...)`
- **Purpose:** Printf-like formatted output to screen with support for %d, %s, %u, %x format specifiers.
- **Inputs:** Format string and variadic arguments.
- **Outputs/Return:** Character count output (or EOF if format ends with orphan '%').
- **Side effects:** Outputs to video memory via helper functions. Initializes cursor to (0, 0) via mysetxy.
- **Calls:** mysetxy, va_start, va_end, printnum, printstring, printunsigned.
- **Notes:** **CRITICAL BUG:** Early `return(0)` statement (marked "DEBUG") at function entry causes all format processing code to be unreachable dead code. Function always returns 0 without output.

## Control Flow Notes

This module forms the **text output layer** of the display pipeline:
1. **Low-level drawing** (DrawText): atomic cell writes to video memory.
2. **Mid-level buffering** (myputch, printstring): character accumulation with cursor advance and scrolling logic.
3. **High-level formatting** (myprintf): variadic argument dispatch (non-functional due to early return).

Expected initialization: caller must set cursor via mysetxy before text output.

## External Dependencies

- `<stdio.h>, <stdarg.h>, <stdlib.h>`: standard C headers.
- `itoa(int, char*, int)` (stdlib): convert signed int to string.
- `ultoa(unsigned long, char*, int)` (stdlib): convert unsigned long to string.
- `myprint.h`: local header (COLORS enum, frame type macros, function declarations).
