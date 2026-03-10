# audiolib/source/user.c

## File Purpose
Provides command-line argument parsing utilities for detecting parameters and retrieving associated values. Supports DOS-style parameter syntax (`-param` or `/param`). Intended for engine initialization and configuration from the command line.

## Core Responsibilities
- Parse command-line arguments (`_argc` / `_argv`)
- Check for presence of flags prefixed with `-` or `/`
- Retrieve text values following parameter flags
- Case-insensitive parameter matching using `stricmp`

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_argc` | `int` | global (extern) | Argument count from C runtime |
| `_argv` | `char**` | global (extern) | Argument vector from C runtime |

## Key Functions / Methods

### USER_CheckParameter
- **Signature:** `int USER_CheckParameter(const char *parameter)`
- **Purpose:** Determines whether a named parameter flag exists on the command line.
- **Inputs:** `parameter` — the parameter name to search for (without `-` or `/` prefix)
- **Outputs/Return:** `TRUE` (1) if found, `FALSE` (0) if not found
- **Side effects:** None; read-only iteration over command-line arguments
- **Calls:** `stricmp()` (string comparison)
- **Notes:** Skips `_argv[0]` (program name); only recognizes arguments starting with `-` or `/`; parameter matching is case-insensitive

### USER_GetText
- **Signature:** `char *USER_GetText(const char *parameter)`
- **Purpose:** Retrieves the value/text immediately following a named parameter on the command line.
- **Inputs:** `parameter` — the parameter name to search for (without `-` or `/` prefix)
- **Outputs/Return:** Pointer to the next argument if parameter found, otherwise `NULL`
- **Side effects:** None; read-only iteration over arguments
- **Calls:** `stricmp()` (string comparison)
- **Notes:** Assumes the parameter is immediately followed by its value; no bounds checking on `_argv[i+1]`; returns `NULL` if parameter not found or if it is the last argument (would access invalid memory)

## Control Flow Notes
Initialization/configuration phase: these functions are typically called early during engine startup to detect user-provided options affecting audio system initialization (e.g., `-nodblbuf`, `-noems`). No frame or render integration.

## External Dependencies
- **Standard includes:** `<dos.h>`, `<string.h>`
- **External globals:** `_argc`, `_argv` (C runtime)
- **Defined elsewhere:** `stricmp()` (case-insensitive string comparison; likely from libc or DOS libraries)
