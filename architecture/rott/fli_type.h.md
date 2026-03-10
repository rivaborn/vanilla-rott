# rott/fli_type.h

## File Purpose
Portable integer type definitions and platform abstraction for the ROTT engine. Provides standardized type names for cross-compiler compatibility and defines boolean/error code semantics. Based on Jim Kent's original Types.h design.

## Core Responsibilities
- Define portable 8/16/32-bit integer type aliases (Char, Uchar, Short, Ushort, Long, Ulong)
- Standardize Boolean and ErrCode semantic types
- Define FileHandle abstraction
- Supply TRUE/FALSE and error code constants (Success, AError)
- Provide compiler/platform macro overrides (int86, inportb, outportb)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Char | typedef | Signed 8-bit integer |
| Uchar | typedef | Unsigned 8-bit integer |
| Short | typedef | Signed 16-bit integer |
| Ushort | typedef | Unsigned 16-bit integer |
| Long | typedef | Signed 32-bit integer |
| Ulong | typedef | Unsigned 32-bit integer |
| Boolean | typedef | Boolean value (int-based) |
| ErrCode | typedef | Error code return type |
| FileHandle | typedef | OS file handle abstraction |

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
Header-only file; no runtime control flow. Included by compilation units requiring portable type definitions. Include guard prevents multiple inclusion.

## External Dependencies
None; self-contained type definitions for engine-wide use.
