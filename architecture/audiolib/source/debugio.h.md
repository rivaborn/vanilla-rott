# audiolib/source/debugio.h

## File Purpose
Debug I/O interface header providing character and string output functions. Declares a small set of utilities for formatted debug output to an unspecified device or buffer, likely used during development and runtime debugging of the audio library.

## Core Responsibilities
- Set cursor position for debug output (DB_SetXY)
- Write individual characters to debug output (DB_PutChar)
- Output strings and numbers in various formats (DB_PrintString, DB_PrintNum, DB_PrintUnsigned)
- Provide printf-style formatted output interface (DB_printf)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### DB_SetXY
- Signature: `void DB_SetXY(int x, int y)`
- Purpose: Set cursor position for subsequent debug output
- Inputs: x, y coordinates
- Outputs/Return: void
- Side effects: Modifies debug output cursor position
- Calls: Not visible from header
- Notes: Implementation elsewhere; coordinates likely map to console or framebuffer positions

### DB_PutChar
- Signature: `void DB_PutChar(char ch)`
- Purpose: Output a single character to debug output
- Inputs: Character to output
- Outputs/Return: void
- Side effects: Writes to debug output stream/device
- Calls: Not visible from header
- Notes: Fundamental building block for higher-level output functions

### DB_PrintString
- Signature: `int DB_PrintString(char *string)`
- Purpose: Output a null-terminated string
- Inputs: String pointer
- Outputs/Return: Count of characters printed (inferred)
- Side effects: Writes string to debug output
- Calls: Not visible from header

### DB_PrintNum
- Signature: `int DB_PrintNum(int number)`
- Purpose: Output a signed integer in decimal
- Inputs: Integer value
- Outputs/Return: Count of characters printed (inferred)
- Side effects: Writes number to debug output
- Calls: Not visible from header

### DB_PrintUnsigned
- Signature: `int DB_PrintUnsigned(unsigned long number, int radix)`
- Purpose: Output unsigned long in arbitrary radix (binary, octal, decimal, hex)
- Inputs: Number, radix (base for conversion)
- Outputs/Return: Count of characters printed (inferred)
- Side effects: Writes formatted number to debug output
- Calls: Not visible from header

### DB_printf
- Signature: `int DB_printf(char *fmt, ...)`
- Purpose: Printf-style formatted output with variadic arguments
- Inputs: Format string and variadic arguments
- Outputs/Return: Count of characters printed (inferred)
- Side effects: Writes formatted output to debug output stream
- Calls: Not visible from header
- Notes: Standard C format string semantics; likely wrapper around platform-specific printf

## Control Flow Notes
Not inferable from header. These functions are debug utilities called ad-hoc throughout initialization, diagnostics, and runtime (not part of main frame/render loop).

## External Dependencies
- Standard C variadic macro system (for DB_printf)
- Implementation defined elsewhere (debugio.c or platform-specific code)
