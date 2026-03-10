# audiolib/source/user.c — Enhanced Analysis

## Architectural Role

This module provides the foundational command-line argument parsing layer for the audio subsystem initialization. It sits at the boundary between the OS environment (C runtime `_argc`/`_argv`) and the audio device drivers (BLASTER, ADLIB, FM, GUS, AWE32). Early in startup, higher-level audio init code (likely in BLASTER or top-level game initialization) calls these functions to detect user directives (e.g., `-nosound`, `-noems`, `-adlib`) that configure which audio devices to initialize or skip.

## Key Cross-References

### Incoming (who depends on this file)
- **Game main loop** (`rott/rt_main.c`): Likely calls `USER_CheckParameter` during initialization phase to detect audio-related flags before calling audio subsystem `Init` functions (BLASTER_Init, ADLIB_Init, AL_Init, etc.)
- **Audio device initialization layers** (BLASTER, ADLIB, FM detection code): May call these utilities to parse device-specific flags
- No explicit references found in the provided cross-reference index, suggesting either minimal public exposure or that call sites predate this documentation

### Outgoing (what this file depends on)
- **C runtime**: `_argc`, `_argv` globals (external); `stricmp()` (case-insensitive string comparison, likely from libc or DOS C runtime)
- **No other audiolib dependencies**: This is a leaf utility with zero coupling to the audio device drivers, keeping it isolated and reusable

## Design Patterns & Rationale

**Linear search + case-insensitive matching**: The O(n) iteration through argv is sufficient for typical command-line sizes (10–50 arguments). Case-insensitivity (`stricmp`) is a DOS/early Windows convention where command-line conventions were inconsistent (`-ADLIB`, `-adlib`, `-AdLib` all accepted). This reduces user friction.

**Prefix filtering (`-` or `/`)**: Supports both Unix-like (`-param`) and DOS-style (`/param`) conventions, reflecting cross-platform heritage of Apogee Software's codebase (DOS, but with some portability in mind).

**Stateless, pure functions**: No side effects or global state mutation. Functions are idempotent and thread-safe (though threading wasn't relevant in 1994). This keeps them suitable for early initialization before engine subsystems spin up.

## Data Flow Through This File

```
_argc, _argv (C runtime)
    ↓
USER_CheckParameter / USER_GetText
    ↓ (linear scan with case-insensitive matching)
    ↓
boolean found  OR  char* text_value
    ↓
Audio device Init code consumes result
    ↓
Audio subsystem initialization (conditional on parameters)
```

Example: `if (USER_CheckParameter("adlib")) { ADLIB_Init(...) }` or `const char *device = USER_GetText("device")` to read device port strings.

## Learning Notes

**Era-appropriate design**: This reflects 1990s game engine initialization patterns—no argument parsing libraries, no structured configuration, pure procedural C. Modern engines would use JSON config files or structured argument parsing (argparse, getopt).

**No input validation**: The caller bears full responsibility for knowing correct parameter formats. `USER_GetText` returns unvalidated `char*`, exposing all downstream code to potential NULL dereference or malformed strings. This is typical of era constraints (no exceptions, minimal error handling culture in C).

**Linear search acceptable then**: With typical argv sizes (<50 args), O(n) parsing is negligible. Modern engines might build a hash table, but the overhead isn't justified here.

**DOS/Windows parameter style**: The dual support for `-` and `/` is a legacy accommodation; modern game engines dropped `/` in favor of POSIX conventions.

## Potential Issues

- **Buffer overrun in `USER_GetText` (line ~105)**: Accessing `_argv[i+1]` without checking `i+1 < _argc` means if a parameter flag is the last argument (e.g., `game -adlib`), the function reads one past the argv array boundary, risking undefined behavior.
- **No validation of returned pointer**: Callers must null-check the result before dereferencing. Easy to miss in historical C codebases.
- **Case-insensitivity hides typos**: User typos like `-ADLB` (instead of `-ADLIB`) silently fail instead of warning.
