# audiolib/source/standard.h

## File Purpose
Standard definitions header for the audio library. Provides common type aliases, utility macros for bit operations and array bounds checking, error code enumerations, and debugging utilities used throughout the codebase.

## Core Responsibilities
- Define standard type aliases (`boolean`, `errorcode`)
- Define standard error codes (`Success`, `Warning`, `FatalError`)
- Provide utility macros for bitwise operations, array handling, and loops
- Provide conditional debugging macros

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| boolean | typedef | Integer-based boolean type |
| errorcode | typedef | Integer-based error code type |
| STANDARD_ERRORS | enum | Standard error codes (Warning, FatalError, Success) |

## Global / File-Static State
None.

## Key Functions / Methods
None (header-only definitions file).

## Control Flow Notes
Not applicable. This is a utility definitions header with no executable code or control flow.

## External Dependencies
- None. File is self-contained with only standard C preprocessor directives.

## Notes
- True/False conditionally defined to avoid conflicts if already defined elsewhere
- `BITSET()` macro checks if a specific bit flag is set
- `ARRAY_LENGTH()` is a compile-time macro for array size calculation
- `WITHIN_BOUNDS()` validates array index safety
- `FOREVER` macro provides infinite loop syntax
- `DEBUG_CODE` macro conditionally compiles debug-only blocks based on `NDEBUG` preprocessor flag
- Error codes follow a standard convention: negative values for errors, zero for success
