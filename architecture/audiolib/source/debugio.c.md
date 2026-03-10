# audiolib/source/debugio.c

## File Purpose
Legacy DOS-era debug output library that writes directly to video memory (monochrome adapter at 0xb0000). Provides character, string, and formatted printing to an on-screen debug console with cursor positioning and auto-scrolling.

## Core Responsibilities
- Direct memory I/O to VGA/MDA display buffer (0xb0000)
- Cursor positioning and line management
- Screen scrolling when display buffer is full
- Character output with printable ASCII filtering
- Number-to-string conversion (decimal, hex, unsigned)
- Printf-style variadic formatted output with %d, %s, %u, %x specifiers

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `disp_offset` | `unsigned short` | static | Tracks current cursor position in video memory (in bytes from 0xb0000) |
| `myutoa` | function pointer | static | Helper: unsigned integer to ASCII string conversion |
| `myitoa` | function pointer | static | Helper: signed integer to ASCII string conversion |

## Key Functions / Methods

### DB_SetXY
- Signature: `void DB_SetXY(int x, int y)`
- Purpose: Set cursor position on debug screen
- Inputs: `x` (column 0–79), `y` (row 0–24)
- Outputs/Return: None
- Side effects: Updates `disp_offset` global state
- Calls: None
- Notes: Assumes 80×24 text mode; calculation `(x * 2) + (y * 160)` accounts for 2-byte per char (char + attribute byte)

### DB_PutChar
- Signature: `void DB_PutChar(char ch)`
- Purpose: Write single character to current cursor position and update cursor
- Inputs: `ch` (character to output)
- Outputs/Return: None
- Side effects: Modifies video memory at 0xb0000; updates `disp_offset`; scrolls screen if at bottom; clears lines on wrap
- Calls: None (direct memory access)
- Notes: Ignores control chars < 32; handles `\r` (carriage return, reset to column 0), `\n` (newline + clear line); auto-scrolls when `disp_offset >= 160*24`

### DB_PrintString
- Signature: `int DB_PrintString(char *string)`
- Purpose: Output null-terminated string character by character
- Inputs: `string` (pointer to null-terminated C string)
- Outputs/Return: Character count printed
- Side effects: Calls `DB_PutChar` for each character
- Calls: `DB_PutChar`
- Notes: Returns count including control characters processed

### DB_PrintNum
- Signature: `int DB_PrintNum(int number)`
- Purpose: Convert signed integer to decimal and print
- Inputs: `number` (signed int)
- Outputs/Return: Character count printed
- Side effects: Allocates 100-byte temporary buffer on stack
- Calls: `myitoa`, `DB_PrintString`
- Notes: None

### DB_PrintUnsigned
- Signature: `int DB_PrintUnsigned(unsigned long number, int radix)`
- Purpose: Convert unsigned integer to string with arbitrary radix and print
- Inputs: `number` (unsigned long), `radix` (base: typically 10 or 16)
- Outputs/Return: Character count printed
- Side effects: Allocates 100-byte temporary buffer on stack
- Calls: `myutoa`, `DB_PrintString`
- Notes: Supports radix 2–36; A–F used for digits > 9

### DB_printf
- Signature: `int DB_printf(char *fmt, ...)`
- Purpose: Printf-like formatted output to debug screen
- Inputs: `fmt` (format string with %d, %s, %u, %x specifiers), variadic args
- Outputs/Return: Total character count printed
- Side effects: Processes variadic arguments; calls formatting helpers
- Calls: `DB_PrintNum`, `DB_PrintString`, `DB_PrintUnsigned`, `DB_PutChar`
- Notes: Returns `EOF` on premature format string termination; only %d, %s, %u, %x supported; case-insensitive %X (hex)

## Control Flow Notes
Initialization-free; all state (`disp_offset`) has static storage duration with default initialization. Debug output is on-demand via public API calls. Scrolling/wrapping handled transparently in `DB_PutChar`. No shutdown or cleanup required.

## External Dependencies
- **Includes**: `<stdio.h>` (EOF), `<stdarg.h>` (va_list, va_start, va_end), `<stdlib.h>` (included but unused), `"debugio.h"` (public API declarations)
- **Direct memory access**: Hardcoded to VGA monochrome display buffer at `0xb0000`
- **External symbols**: None; all helper functions defined locally (static)
