# audiolib/source/assert.h

## File Purpose
Debug assertion macro library for runtime condition checking. Provides a conditional assertion facility that calls an error reporting function on failure in debug builds and compiles to nothing in release builds (NDEBUG mode).

## Core Responsibilities
- Define `ASSERT` macro for condition checking in debug mode
- Declare external assertion failure handler (`_Assert`)
- Support Watcom C++ compiler conventions via pragma directives
- Provide guard against multiple header inclusions

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### _Assert
- Signature: `void _Assert(char *strFile, unsigned uLine)`
- Purpose: Handle assertion failure by reporting the file and line number where assertion failed
- Inputs: `strFile` (file name as string), `uLine` (line number as unsigned integer)
- Outputs/Return: Does not return (marked with `#pragma aux _Assert aborts`)
- Side effects: Likely terminates program execution or triggers debugger break
- Calls: Not defined in this file (external implementation)
- Notes: Watcom-specific pragma declares this function as non-returning

### ASSERT macro
- Signature: `ASSERT(f)` where `f` is a condition expression
- Purpose: Evaluate condition `f` at runtime; call `_Assert` if condition is false
- Inputs: `f` — any evaluable C expression (integer, pointer, etc.)
- Outputs/Return: None (macro expands to statement)
- Side effects: Calls `_Assert` with `__FILE__` and `__LINE__` if condition fails
- Calls (expands to): `_Assert(__FILE__, __LINE__)`
- Notes: 
  - Uses double-negation pattern (`if (f) ; else _Assert(...)`)
  - Completely removed from compiled code when `NDEBUG` is defined (release builds)
  - Guard check (`#else` branch) prevents double-inclusion errors

## Control Flow Notes
The macro is activated only in debug builds (when `NDEBUG` is not defined). On assertion failure, it invokes the non-returning `_Assert` function, which halts execution. In release builds, the macro expands to nothing, resulting in zero runtime overhead.

## External Dependencies
- `__FILE__`, `__LINE__` preprocessor constants (C standard)
- `#pragma aux` directive (Watcom C/C++ compiler-specific)
- External symbol: `_Assert` (defined elsewhere in audiolib)
