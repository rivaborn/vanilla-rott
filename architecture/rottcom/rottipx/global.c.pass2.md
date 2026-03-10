# rottcom/rottipx/global.c — Enhanced Analysis

## Architectural Role

This file provides the utility layer for the IPX networking setup subsystem. It serves the initialization and error-handling pipeline of `rottipx`, coordinating between command-line parameter discovery (`CheckParm`) and safe teardown (`Error` → `Shutdown`). The duplication of `CheckParm` across IPX and serial (`rottser`) subsystems reflects a modular architecture where networking modes (IPX vs. serial) are independently implemented with their own utility stacks.

## Key Cross-References

### Incoming (who depends on this file)
- The IPX setup initialization likely calls `CheckParm` to discover networking parameters (similar pattern in `rottcom/rottser/global.c`)
- Error conditions in `ipxsetup.h` logic trigger `Error()` to ensure `Shutdown()` before exit
- Functions are exported via `rottcom/rottipx/global.h` (header inclusion pattern)

### Outgoing (what this file depends on)
- **`Shutdown()`** from `ipxsetup.h` — IPX-specific resource cleanup (memory, network state)
- **`_argc` / `_argv`** — C runtime globals from main entry point
- **`stricmp()`** — DOS/legacy case-insensitive comparison (string.h)

## Design Patterns & Rationale

**Utility Duplication Pattern:**
`CheckParm` is defined independently in both IPX (`rottcom/rottipx/global.c`) and serial (`rottcom/rottser/global.c`) subsystems, also appearing in `rtsmaker/cmdlib.c` and `rott/rt_util.c`. This duplication reflects **no centralized utility library** — each networking subsystem was self-contained, likely for link-time modularity in a DOS environment. Modern engines would consolidate this into a shared library.

**Error Handling as Panic Gate:**
The `Error()` function enforces a **mandatory shutdown contract**: callers cannot bypass `Shutdown()` when terminating abnormally. The commented code (`exit(1)` disabled, "Clean exit from SERSETUP") suggests this was refined during debugging to always call `Shutdown()` regardless of error type.

## Data Flow Through This File

1. **Initialization Phase**: `CheckParm` scans `_argc`/`_argv` (filled by C runtime's main), returns argument position if found
2. **Runtime**: Setup code uses returned position to read configuration
3. **Error/Shutdown Phase**: Any abnormal condition triggers `Error(error, ...)`, which:
   - Formats and prints the error message (variadic args)
   - Calls `Shutdown()` to release IPX resources (sockets, buffers, etc.)
   - Exits with code 1 (error!=NULL) or 0 (error==NULL)

## Learning Notes

**Idiomatic to this era:**
- **No dynamic allocation abstractions** — `_argc`/`_argv` directly from C runtime
- **DOS-era string handling** — `stricmp()` over modern UTF-8 aware comparisons
- **No logging framework** — errors print directly to stdout via `vprintf()`
- **Manual shutdown contracts** — functions like `Shutdown()` must be called explicitly; no RAII or cleanup handlers

**Contrasts with modern practice:**
- Modern engines use dependency injection or lifecycle managers to ensure cleanup
- Command-line parsing is typically centralized (argparse, Click, etc.)
- Error handling separates formatting from execution (log != exit)

**Connection to engine architecture:**
This file is part of the **networking setup layer** — one of multiple isolated subsystems (IPX, modem, direct link) that are plugged in at startup. The duplication of `CheckParm` across subsystems is a precursor to plugin architectures, where each module owns its initialization.

## Potential Issues

- **Resource leak if `Shutdown()` throws**: No exception safety; if `Shutdown()` crashes, cleanup may be incomplete
- **Commented code suggests incomplete refactoring**: The disabled `exit(1)` and SERSETUP comment may indicate unfinished migration between setup utilities
- **No arg bounds checking**: `CheckParm` iterates from 1 to argc-1 safely, but callers must validate argv[index] exists
