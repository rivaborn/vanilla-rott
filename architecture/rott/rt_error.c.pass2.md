# rott/rt_error.c — Enhanced Analysis

## Architectural Role

This module is a **critical cross-cutting concern** providing DOS-level exception handling for the ROTT engine. It sits at the boundary between the OS/hardware and the game loop, intercepting hard disk errors (via DOS INT 0x24) and CPU division-by-zero faults (INT 0x00) before they crash the game. As a lifecycle-managed subsystem (startup/shutdown pairing), it initializes before the game loop and uninstalls handlers on exit, ensuring OS stability. The module bridges the OS error reporting layer with the engine's UI system, translating opaque DOS error codes into player-facing dialogs with retry/abort choices.

## Key Cross-References

### Incoming (who depends on this file)
- **Main engine startup/shutdown** (inferred): `UL_ErrorStartup()` and `UL_ErrorShutdown()` are called during engine initialization and teardown (likely from `engine.c` or `rt_main.c`).
- **Main game loop** (inferred): Checks the global `DivisionError` flag during frame updates to detect division-by-zero faults after they occur (ISR sets flag, loop polls it).
- **Keyboard subsystem** (rt_isr / rt_input): Reads `Keyboard[]` array state and `KeyboardStarted` flag to determine input method (polled vs. buffered keyboard state).

### Outgoing (what this file depends on)
- **Video subsystem** (`rt_vid.h`, `modexlib.h`): Calls `SetBorderColor()`, `VL_Bar()`, `US_CPrint()`, reads/writes CRTC registers via `OUTP()` macro. Queries video mode via raw memory address `0x449`.
- **String/UI subsystem** (`rt_str.h`, `rt_menu.h`): Calls `US_MeasureStr()`, uses `CurrentFont`, `tinyfont`, `PrintX`, `PrintY` globals.
- **Keyboard subsystem** (`isr.h`): Reads global `Keyboard[]` array and `KeyboardStarted` flag.
- **Fatal error handler** (`rt_util.h` or elsewhere): Calls `Error()` if user aborts a disk operation (fatal exit).
- **DOS/DJGPP runtime** (`<dos.h>`): `_harderr()` (install hard error handler), `_dos_getvect()`, `_dos_setvect()` (interrupt vector manipulation).
- **Standard C** (`<conio.h>`, `<stdio.h>`): `kbhit()`, `getch()`, `printf()`, `toupper()`, `vsprintf()`.

## Design Patterns & Rationale

### Interrupt Handler Registration Pattern
The module uses **idempotent lifecycle management** with guards (`ErrorHandlerStarted` flag) to prevent double-installation of handlers. `UL_StartupDivisionByZero()` saves the old INT 0x00 vector and installs the new ISR; `UL_ShutdownDivisionByZero()` restores the old vector. This pattern is essential for DOS/embedded systems where interrupt vectors are scarce resources.

### Graceful Degradation (Dual UI Fallback)
`UL_UserMessage()` checks video mode `0x449` and either:
- **Renders to VGA graphics mode**: windowed box with `VL_Bar()`, text via `US_CPrint()`, CRTC register manipulation for page flipping.
- **Falls back to console**: `printf()` for environments without VGA or during system shutdown.

This reflects early 90s game development constraints: support both modern VGA systems and legacy text-only boot scenarios.

### Error Code Decoding (Bit-Field Extraction)
DOS error codes are packed into a single `unsigned` word with bit flags (DISKERROR, LOCATION, READWRITEERROR, etc.). The handler extracts these via **bitmask macros**, mapping error numbers to human-readable strings. This compact encoding was typical of DOS-era APIs.

### Flag-Based Exception Continuation
Rather than traditional exception handling, `UL_DivisionISR()` sets a global `DivisionError` flag and returns. The **main game loop polls this flag** during frame updates. This is a polling/continuation model, not true exception propagation—common in systems where throwing across ISRs is risky or unsupported.

### Hard Error Handler Always Returns RETRY
`UL_harderr()` always returns `_HARDERR_RETRY` regardless of user choice (abort or retry). If user chooses abort, the function calls fatal `Error()` directly, terminating before returning. This avoids ambiguity in DOS error handling semantics.

## Data Flow Through This File

```
┌─────────────────────────────────────────────────────────────┐
│ Disk/Device Error Path                                      │
├─────────────────────────────────────────────────────────────┤
│ Hardware error → DOS INT 0x24 → UL_harderr()               │
│                  ↓                                           │
│             Decode deverr bits (DISKERROR, LOCATION, RW)    │
│                  ↓                                           │
│             UL_GeneralError() or UL_DriveError()            │
│                  ↓                                           │
│             UL_UserMessage() (render dialog)                │
│                  ↓                                           │
│             Poll keyboard (Keyboard[] or kbhit/getch)       │
│                  ↓                                           │
│             User presses (A)bort or (R)etry                │
│                  ↓                                           │
│             If abort: call Error() (fatal exit)             │
│             If retry: return _HARDERR_RETRY to DOS          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Division-by-Zero Path                                       │
├─────────────────────────────────────────────────────────────┤
│ CPU divide-by-zero → INT 0x00 → UL_DivisionISR()           │
│                      ↓                                       │
│                 DivisionError = true                        │
│                 SetBorderColor() (visual debug indicator)    │
│                 OUTP(0x20, 0x20) (EOI to interrupt ctrl)    │
│                 Return to calling instruction                │
│                      ↓                                       │
│                 Main game loop checks DivisionError flag     │
│                 (during next frame update)                  │
│                      ↓                                       │
│                 Engine logs/handles gracefully or aborts     │
└─────────────────────────────────────────────────────────────┘
```

## Learning Notes

### Era-Specific Patterns
This module exemplifies **1990s DOS game development** constraints:
- **Raw interrupt vectors**: Direct manipulation of CPU interrupt tables; modern engines use OS exception APIs (Windows SEH, POSIX signals).
- **Hard-coded memory addresses** (`0x449` for video mode): Direct hardware polling instead of OS abstraction layers.
- **Packed error codes**: DOS error word compressed with bitmasks; modern OSes use structured error objects or enums.
- **Text-only fallback UI**: Reflects support for legacy environments and boot-sector compatibility.

### Idiomatic Game Engine Decisions
- **No exception throwing in ISRs**: Flag-based continuation avoids C++ exception safety issues in interrupt context.
- **Keyboard state polling**: Reads global `Keyboard[]` array (likely built by a separate ISR-driven handler) rather than blocking I/O; asynchronous game loop design.
- **Graceful degradation**: UI fallback shows testing against multiple video modes and environments.

### Connections to Modern Engine Concepts
- **Cross-cutting concern**: Error handling is **orthogonal to core gameplay** (actor, map, render systems), injected at the boundary.
- **Lifecycle management**: Startup/shutdown pairing mirrors modern engine plugin systems (initialize → service → deinitialize).
- **State machine for error recovery**: Retry/abort dialog is a simple state machine (waiting → user input → action).

## Potential Issues

1. **Hardcoded Video Mode Check** (line 125: `if (*(byte *)0x449 == 0x13)`):
   - Directly dereferences BIOS data segment address (video mode byte). Non-portable; breaks on modern systems or under VM/debuggers where this address may not be accessible.
   - No bounds checking; UB if address is invalid.

2. **Incomplete Error Decoding**:
   - `whichDrive` is **never extracted from `deverr`** (line 272: `int whichDrive = 0;` is initialized but never assigned from the deverr bitmask). Always passes 0 (drive A) to `UL_DriveError()`.
   - This likely a **bug**: disk errors on drives B–G will incorrectly report drive A.

3. **Hard Error Handler Behavior Opacity**:
   - `UL_harderr()` returns `_HARDERR_RETRY` unconditionally. If the user chooses abort, `Error()` is called (fatal exit), so the return never executes. This is correct but non-obvious; a developer might assume the return value is meaningful.

4. **Signal Safety**:
   - `UL_DivisionISR()` calls `SetBorderColor()` (likely a function call), not just setting a flag. ISRs should minimize stack usage and function calls; any re-entrancy or state corruption in `SetBorderColor()` would corrupt the faulting context.

5. **Memory Hardcoding**:
   - `colormap + (((100-10)>>2)<<8) + 160` is a magic offset into the colormap (likely palette). No validation; if colormap is NULL or undersized, this reads uninitialized memory.

6. **No DOS Version Check**:
   - Code assumes DOS error interrupt conventions are available. Modern DOSBox emulation or DJGPP under Windows/Linux may not support these exactly as expected.
