# rottcom/rottipx/global.h

## File Purpose
Global header providing common type definitions, macros, and utility declarations for the Rise of the Triad engine. Establishes platform abstraction layer for I/O operations, boolean types, and system constants.

## Core Responsibilities
- Define portable type aliases (byte, WORD, LONG, boolean)
- Provide hardware I/O abstractions (port input/output, interrupt control) for DOS/x86 compatibility
- Define common constants (TRUE/FALSE, EOS, ESC, clock frequency)
- Declare global utility functions (error reporting, parameter checking)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| boolean | enum | Custom boolean type with values {false, true} |
| byte, BYTE | typedef | Unsigned 8-bit integer |
| WORD | typedef | Unsigned 16-bit integer |
| LONG | typedef | Unsigned 32-bit integer |

## Global / File-Static State
None.

## Key Functions / Methods

### Error
- Signature: `void Error (char *error, ...);`
- Purpose: Report fatal errors to the user
- Inputs: Format string and variadic arguments (printf-style)
- Outputs/Return: None (void)
- Side effects: Likely terminates execution after displaying error message
- Calls: Not inferable from this file
- Notes: Variadic function; implementation defined elsewhere

### CheckParm
- Signature: `int CheckParm (char *check);`
- Purpose: Check for presence of a command-line parameter
- Inputs: Parameter string to search for
- Outputs/Return: Integer (likely boolean: 0 = not found, non-zero = found/index)
- Side effects: Not inferable from this file
- Calls: Not inferable from this file
- Notes: Implementation defined elsewhere

## Control Flow Notes
This is a header-only file providing compile-time definitions. Included early in translation units to establish type system and platform abstractions. Does not directly participate in runtime control flow.

## External Dependencies
- **Defined elsewhere**: `inp()`, `outp()`, `disable()`, `enable()` — DOS/x86 I/O and interrupt control functions (platform-specific library)
- **No explicit includes shown** — relies on platform headers providing the above functions
