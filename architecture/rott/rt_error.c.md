# rott/rt_error.c

## File Purpose
Manages DOS-level error handling for the ROTT engine, including hard disk errors, device errors, and division-by-zero exceptions. Provides user-facing error dialogs with retry/abort options and installs interrupt service routines for critical hardware errors.

## Core Responsibilities
- Install and manage DOS hard error handler (`_harderr`)
- Intercept and handle division-by-zero (INT 0x00) exceptions
- Display formatted error messages in a windowed UI or via console
- Parse DOS device error codes and present human-readable error details
- Allow user to retry or abort operations on disk/device errors
- Maintain error handler startup/shutdown lifecycle

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ErrorCodes[13][25]` | static char array | Maps DOS error codes (0–12) to descriptive messages |
| `Drives[7][3]` | static char array | Drive letter labels (A–G) |
| `Locations[4][11]` | static char array | Error location labels (MS-DOS, FAT, Directory, Data area) |
| `ReadWrite[2][6]` | static char array | Read/Write operation labels |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DivisionError` | boolean | global | Flag set when division-by-zero occurs; checked by main engine |
| `ErrorHandlerStarted` | boolean | static | Guards against redundant handler installation |
| `olddivisr` | void (__interrupt __far *)() | static | Saved pointer to previous INT 0x00 handler; restored on shutdown |

## Key Functions / Methods

### UL_UserMessage
- **Signature:** `void UL_UserMessage (int x, int y, char *str, ...)`
- **Purpose:** Display a formatted message in a window (if video mode 0x13) or via `printf`.
- **Inputs:** `x, y` (window position), `str` (format string), variable args.
- **Outputs/Return:** None.
- **Side effects:** Modifies video memory (displayofs, bufferofs); changes CurrentFont, PrintX, PrintY globals; calls VL_Bar, US_CPrint, or printf.
- **Calls:** `memset`, `va_start/va_end`, `vsprintf`, `*(byte *)0x449` (check video mode), `US_MeasureStr`, `VL_Bar`, `US_CPrint`, `OUTP` (CRTC register writes), `printf`.
- **Notes:** Handles both VGA graphics mode and text console fallback. Uses tinyfont and fixed window dimensions.

### UL_GeneralError
- **Signature:** `int UL_GeneralError (int code)`
- **Purpose:** Display a general device error and wait for user to choose (A)bort or (R)etry.
- **Inputs:** `code` — index into `ErrorCodes` array.
- **Outputs/Return:** 1 if user chooses Abort, 0 if Retry.
- **Side effects:** Calls `UL_UserMessage`; polls keyboard state via `Keyboard[]` array or `kbhit()/getch()`.
- **Calls:** `UL_UserMessage`, `toupper`, `getch`, `kbhit`.
- **Notes:** Keyboard handler is checked; if not started, falls back to `kbhit/getch`. Waits for key release before accepting next input.

### UL_DriveError
- **Signature:** `int UL_DriveError (int code, int location, int rwerror, int whichdrive)`
- **Purpose:** Display a drive-specific error with location and read/write details; await user choice.
- **Inputs:** `code` (ErrorCodes index), `location` (Locations index), `rwerror` (ReadWrite index), `whichdrive` (Drives index).
- **Outputs/Return:** 1 if Abort, 0 if Retry.
- **Side effects:** Same as `UL_GeneralError` (keyboard polling, `UL_UserMessage` call).
- **Calls:** `UL_UserMessage`, `toupper`, `getch`, `kbhit`.
- **Notes:** More detailed error reporting than `UL_GeneralError`; includes drive letter and error location.

### UL_harderr
- **Signature:** `int __far UL_harderr (unsigned deverr, unsigned errcode, unsigned far *devhdr)`
- **Purpose:** DOS hard error interrupt handler; parses device error flags and invokes appropriate user dialog.
- **Inputs:** `deverr` (device error code with bit flags), `errcode` (error code), `devhdr` (device header pointer).
- **Outputs/Return:** Always returns `_HARDERR_RETRY` to signal retry to DOS.
- **Side effects:** Calls `UL_GeneralError` or `UL_DriveError`; may call `Error()` if user aborts.
- **Calls:** Bitwise extraction macros (DISKERROR, IGNOREAVAILABLE, LOCATION, READWRITEERROR), `UL_GeneralError`, `UL_DriveError`, `Error`.
- **Notes:** Decodes DOS error flags (`deverr`) using bitmasks defined at top of file. If user aborts, calls fatal `Error()`. Always returns _HARDERR_RETRY regardless of user choice.

### UL_DivisionISR
- **Signature:** `void __interrupt __far UL_DivisionISR ( void )`
- **Purpose:** Interrupt service routine for INT 0x00 (division by zero). Sets a flag and acknowledges the interrupt.
- **Inputs:** None (ISR called by CPU).
- **Outputs/Return:** None.
- **Side effects:** Sets global `DivisionError = true`; modifies border color; sends EOI (0x20) to interrupt controller (port 0x20).
- **Calls:** `SetBorderColor`, `OUTP`.
- **Notes:** Uses colormap to set border color based on a fixed offset. EOI signal allows further interrupts. Main engine must check `DivisionError` flag.

### UL_ErrorStartup
- **Signature:** `void UL_ErrorStartup ( void )`
- **Purpose:** Initialize error handling; install hard error and division-by-zero handlers.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Sets `ErrorHandlerStarted` flag; installs `UL_harderr` via `_harderr()`; calls `UL_StartupDivisionByZero()`.
- **Calls:** `_harderr`, `UL_StartupDivisionByZero`.
- **Notes:** Idempotent — returns early if already started.

### UL_ErrorShutdown
- **Signature:** `void UL_ErrorShutdown ( void )`
- **Purpose:** Uninstall error handlers and restore system state.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Clears `ErrorHandlerStarted` flag; calls `UL_ShutdownDivisionByZero()` to restore old INT 0x00.
- **Calls:** `UL_ShutdownDivisionByZero`.
- **Notes:** Idempotent — returns early if not started. Must be called to avoid leaving custom ISRs installed.

### UL_StartupDivisionByZero
- **Signature:** `void UL_StartupDivisionByZero ( void )`
- **Purpose:** Save current INT 0x00 handler and install `UL_DivisionISR`.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Saves old handler to `olddivisr`; installs `UL_DivisionISR` as INT 0x00.
- **Calls:** `_dos_getvect`, `_dos_setvect`.
- **Notes:** Called by `UL_ErrorStartup`. Paired with `UL_ShutdownDivisionByZero`.

### UL_ShutdownDivisionByZero
- **Signature:** `void UL_ShutdownDivisionByZero ( void )`
- **Purpose:** Restore previous INT 0x00 handler.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Restores `olddivisr` to INT 0x00 vector.
- **Calls:** `_dos_setvect`.
- **Notes:** Paired with `UL_StartupDivisionByZero`. Called by `UL_ErrorShutdown`.

## Control Flow Notes
This module operates outside the normal frame/render loop. It installs exception handlers at engine startup (`UL_ErrorStartup`) and uninstalls them at shutdown (`UL_ErrorShutdown`). The DOS hard error handler (`UL_harderr`) and division-by-zero ISR (`UL_DivisionISR`) are event-driven, invoked by the CPU when exceptions occur. The main engine checks the `DivisionError` flag during its update/render phases to detect division-by-zero faults. User errors are displayed via the windowing/UI system (`UL_UserMessage`, `US_CPrint`), which depends on video mode.

## External Dependencies
- **System headers:** `<dos.h>`, `<errno.h>`, `<io.h>`, `<stdio.h>`, `<conio.h>`, `<stdarg.h>`, `<mem.h>`, `<ctype.h>` — DOS/DJGPP primitives.
- **Project headers:** `rt_def.h` (typedefs), `rt_str.h` (string/font functions), `rt_menu.h` (font globals, PrintX/Y), `isr.h` (keyboard state: `Keyboard[]`, `KeyboardStarted`), `rt_vid.h` (video output), `w_wad.h`, `z_zone.h`, `rt_util.h`, `modexlib.h`, `memcheck.h`.
- **External symbols (defined elsewhere):** `SetBorderColor`, `colormap` (from video subsystem), `US_MeasureStr`, `VL_Bar`, `US_CPrint` (from rt_str/rt_menu), `Keyboard[]`, `KeyboardStarted` (from isr), `Error` (fatal error function), `_harderr`, `_dos_getvect`, `_dos_setvect` (DOS/DJGPP).
- **Constants via macros:** `OUTP` (likely port I/O macro from modexlib), `CRTC_*` registers, `MAXKEYBOARDSCAN`.
