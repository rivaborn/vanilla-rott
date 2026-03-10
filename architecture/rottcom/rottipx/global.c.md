# rottcom/rottipx/global.c

## File Purpose
Provides core utility functions for error handling and command-line argument parsing in the IPX networking setup subsystem. The Error function implements abnormal termination with formatted output, while CheckParm enables parameter discovery during initialization.

## Core Responsibilities
- Print formatted error messages with variadic arguments
- Orchestrate clean shutdown before exit
- Locate and return command-line parameter positions
- Support DOS/real-mode runtime environment

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_argc` | int | global (C runtime) | Argument count from main entry |
| `_argv` | char** | global (C runtime) | Argument vector from main entry |

## Key Functions / Methods

### Error
- **Signature:** `void Error(char *error, ...)`
- **Purpose:** Handle abnormal program termination with optional formatted error message
- **Inputs:** Variadic format string + arguments (printf-style)
- **Outputs/Return:** None (calls exit())
- **Side effects:** Calls Shutdown(); prints to stdout; exits process
- **Calls:** `Shutdown()` (ipxsetup.h), `va_start()`, `va_end()`, `vprintf()`, `printf()`, `exit()`
- **Notes:** Comments reference exit(1) and "Clean exit from SERSETUP" but both are disabled. Exit code is 1 if error != NULL, else 0.

### CheckParm
- **Signature:** `int CheckParm(char *check)`
- **Purpose:** Search command-line arguments for a matching parameter (case-insensitive)
- **Inputs:** Parameter string to find
- **Outputs/Return:** Argument index (1 to argc-1) if found; 0 if not found
- **Side effects:** None
- **Calls:** `stricmp()` (string.h)
- **Notes:** Uses DOS-era case-insensitive string comparison; typical usage is initialization parameter validation

## Control Flow Notes
These are utility functions invoked during program initialization (CheckParm) and shutdown (Error). CheckParm is likely called early in setup to detect networking mode or configuration flags; Error serves as a panic handler that ensures clean resource release via Shutdown() before termination.

## External Dependencies
- **Includes:** `<stdarg.h>` (va_list, va_start, va_end), `<stdlib.h>` (exit), `<stdio.h>` (printf, vprintf), `<string.h>` (stricmp), `<dos.h>` (DOS APIs, likely defunct)
- **Defined elsewhere:** `Shutdown()` (ipxsetup.h), `_argc`/`_argv` (C runtime), `socketid`, `server`, `numnetnodes` (ipxsetup.h externs—not used in this file)
