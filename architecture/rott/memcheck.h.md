# rott/memcheck.h

## File Purpose

Memory debugging library header file (MemCheck 3.0 Professional) that intercepts memory allocation/deallocation and string operations to detect buffer overflows, underflows, memory leaks, and invalid pointer usage. Provides cross-compiler DOS support with extensive configuration and callback mechanisms.

## Core Responsibilities

- Define memory tracking data structures (MEMREC) and error codes (MCE_*)
- Detect compiler type and memory model, establish abstraction layer for 16/32-bit code
- Intercept standard C library functions (malloc, free, strcpy, etc.) via macro redefinition
- Declare MemCheck API functions for initialization, checking, and reporting
- Provide callback registration for custom error handling, tracking, and validation
- Support both C and C++ (with overloaded new/delete operators)
- Define link-time configuration macros (MC_SET_*) for compile-time settings

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| MEMREC (struct MemRecord) | struct | Core tracking record: pointer, function ID, size, source file/line, allocation number, flags |
| MCSETTINGS | struct | Runtime configuration: flags, max memory, null check bytes, check byte count, alignment, tracking directory |
| MCCfgInfo | struct | Configuration file format: sentinel + saved MCSETTINGS |
| MCEXCEPTINFO | struct | Exception stack frame (16-bit or 32-bit variant with registers) |
| MCCRITSECT | struct | Critical section lock for multitasking: action, locked flag pointer, reserved |
| ERF | typedef | Error reporting function pointer: `void (*)(char *)` |
| TRACKF | typedef | Tracking callback: `void (*)(int optype, MEMRECP)` on alloc/dealloc |
| CHECKF | typedef | Transfer validation callback: `void (*)(int error, void*, long)` |
| GLOBALF | typedef | Interception callback: `void (*)()` on every runtime function call |
| STARTF / ENDF | typedef | Startup/shutdown callbacks: `void (*)()` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MC_ERF | ERF | extern | Current error reporting function pointer |
| MC_CheckF | CHECKF | extern | Transfer checking callback |
| MC_CritF | MCCRITF | extern | Critical section enter/exit callback |
| MC_GlobalF | GLOBALF | extern | Global interception callback |
| MC_TrackF | TRACKF | extern | Alloc/dealloc tracking callback |
| MC_StartF / MC_EndF | STARTF / ENDF | extern | Startup/shutdown callbacks |
| MC_Settings | MCSETTINGS | extern | Active runtime settings (user should not modify directly) |
| MC_DefaultSettings | MCSETTINGS | extern | Default settings template |
| MC_LogFile | char* | extern | Log file name for error reports |
| MC_UserAutoInit | char | extern | Enable automatic initialization |
| MC_ExceptList[] | unsigned char | extern | Exception numbers to handle (0xFF-terminated) |
| MC_ExceptInfo | MCEXCEPTINFO | extern | Current exception state |
| MC_RocketAllocF / MC_RocketFreeF | function pointers | extern | Disk Rocket allocator functions |

## Key Functions / Methods

When MEMCHECK is defined (active), the following are declared as extern; otherwise defined as no-op macros:

### mc_startcheck
- Signature: `void mc_startcheck(MCSF file, MCSL line, ERF erf)`
- Purpose: Initialize MemCheck, optionally with custom error reporting function
- Inputs: source file name, line number, error reporting function pointer
- Side effects: Initializes internal tracking structures, sets up interception hooks
- Calls: Internal MemCheck initialization
- Notes: Macro expands to include `__FILE__` and `__LINE__`; called at program start or via AutoInit

### mc_endcheck
- Signature: `MCEFLAGS mc_endcheck(void)`
- Purpose: Finalize MemCheck and return accumulated error flags
- Outputs: Bitfield of MCEFLAGS indicating all errors encountered
- Side effects: Flushes logs, reports results, may call user callbacks
- Notes: Macro-wrapped to pass location; typically called at program exit or via atexit()

### mc_check
- Signature: `int mc_check(void _MCFAR *ptr)`
- Purpose: Verify if pointer is within a registered tracked buffer
- Inputs: Heap pointer to validate
- Outputs: Error code (0 if valid, MCE_* if error)
- Calls: Internal buffer lookup, validation
- Notes: Macro-wrapped with location tracking

### mc_register / mc_unregister
- Signature: `void mc_register(void _MCFAR *ptr, unsigned long size)` / `void mc_unregister(void _MCFAR *ptr)`
- Purpose: Manually mark buffers for memory tracking
- Inputs: Pointer and size (for register); pointer only (for unregister)
- Side effects: Adds/removes entry in MemCheck's B-tree database
- Calls: Tracking callback if registered

### mc_check_transfer
- Signature: `int mc_check_transfer(void _MCFAR *src, void _MCFAR *dest, unsigned long size, unsigned srcid, unsigned destid, MEMRECP memrec)`
- Purpose: Validate data transfer operations (memcpy, strcpy, etc.)
- Inputs: Source pointer, destination, size, source/destination function IDs, optional MEMREC
- Outputs: Error code
- Calls: Custom check function callback (CHECKF) if registered
- Notes: Detects overlaps (if MCF_NO_OVERLAP not set), buffer bounds, null pointers

### mc_set_erf / mc_get_erf
- Signature: `ERF mc_set_erf(ERF erf)` / `ERF mc_get_erf(void)`
- Purpose: Install/retrieve error reporting function
- Inputs: Function pointer for mc_set_erf
- Outputs: Previous ERF for mc_set_erf, current ERF for mc_get_erf
- Side effects: Changes error notification behavior

### mc_set_trackf / mc_set_checkf
- Signature: `TRACKF mc_set_trackf(TRACKF tf)` / `CHECKF mc_set_checkf(CHECKF cf)`
- Purpose: Install allocation tracking or transfer checking callbacks
- Inputs: Callback function pointers
- Outputs: Previous callback
- Side effects: Invokes callback on each matching event

### mc_stack_trace
- Signature: `int mc_stack_trace(char *memo)`
- Purpose: Generate and optionally log current call stack
- Inputs: Optional memo string for log
- Outputs: Frame count or error status
- Calls: Stack frame handler callback (SSFH)
- Notes: Useful for debugging when integrated with debugger

### mc_report
- Signature: `void mc_report(REPORTF rf)`
- Purpose: Iterate over all tracked allocations, invoking callback
- Inputs: Report callback function
- Side effects: Calls REPORTF for each MEMREC (plus REPORT_START/REPORT_END markers)

### mc_error_flags / mc_error_text
- Signature: `MCEFLAGS mc_error_flags(void)` / `char* mc_error_text(int error_code)`
- Purpose: Query accumulated errors or get human-readable error description
- Outputs: Error flags bitfield or error string

### mc_set_speed
- Signature: `void mc_set_speed(int speed_mode)`
- Purpose: Switch between MC_RUN_NORMAL and MC_RUN_FAST modes
- Inputs: Speed mode constant
- Notes: Fast mode uses memory-only tracking; normal uses B-tree with optional disk

### Interception Macros (malloc, free, strcpy, memcpy, etc.)
- Purpose: Redirect standard library calls to MemCheck-instrumented versions
- Macro substitution: `#define malloc(_sz) _INTERCEPT(malloc(_sz))`
- Expands to: Location tracking call + actual RTL function call (renamed with `_mc` suffix)
- Calls: `_mcsl()` or `_mcslx()` for location advancement before each call

## Control Flow Notes

**When MEMCHECK is defined (MemCheck active):**
1. **Initialization**: mc_startcheck() called explicitly or via AutoInit flag; sets up B-tree database
2. **Allocation interception**: Each malloc/calloc redirected; location tracked, record added to tree, TRACKF callback invoked
3. **String/memory operations**: strcpy, memcpy, etc. call mc_check_transfer() to validate bounds
4. **Deallocation**: free() checks for double-free, invokes TRACKF, optionally fills buffer with MC_FreedBufferFillChar
5. **Error handling**: ERF callback invoked on each detected error; error flags accumulated globally
6. **Shutdown**: mc_endcheck() reports all leaks, calls ENDF callback, returns error flags
7. **Global interception**: GLOBALF callback can fire on every interception if set

**When NOMEMCHECK is defined:**
- All MemCheck macros evaporate to no-ops or direct library calls
- Zero runtime overhead in production builds

## External Dependencies

- **Compiler detection**: Preprocessor symbols from MSC, Borland, Watcom, Intel, ANSI C compilers
- **Standard headers**: stdio.h (NULL, FILE), stdarg.h (va_list), malloc.h / alloc.h (compiler-specific), string.h
- **Assumed OS**: 16-bit or 32-bit DOS; real mode or protected mode (DPMI, Phar Lap)
- **Defined elsewhere**: RTL replacements (malloc_mc, free_mc, etc.) in MemCheck libraries; exception handlers; stock callbacks (erf_default, trackf_all, etc.)
- **Linker-time**: MC_SET_* macro instantiations in user code link with MemCheck library globals

---

**Notes**: This is a highly sophisticated legacy debugging tool with cross-compiler abstraction, compiler-specific intrinsic disabling pragmas, and careful macro design to preserve source locations through multi-layer interception. The dual-mode design (MEMCHECK vs. NOMEMCHECK) allows zero-overhead production builds. Extensive support for 16-bit segmented memory models (far/near pointers) and multitasking critical sections reflects DOS-era constraints.
