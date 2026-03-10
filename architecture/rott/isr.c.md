# rott/isr.c

## File Purpose
Manages low-level hardware interrupts for timer and keyboard input on DOS systems. Provides ISR hooks for the system timer (PIT) and keyboard controller, maintaining game tick counts and processing raw scan codes into a keyboard event queue.

## Core Responsibilities
- Timer interrupt handling: increments game tick counter (`ticcount`) at ~35 Hz
- Keyboard interrupt handling: reads scan codes, populates keyboard queue, manages shift/extended key states
- Keyboard LED control (caps lock, num lock, scroll lock) via keyboard controller commands
- System timer initialization/shutdown with task-based scheduling fallback
- CMOS time reading for game initialization and profiling
- Delay function for waiting on tick increments

## Key Types / Data Structures
None defined in this file. Uses `task` struct from `task_man.h` for timer task scheduling.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ExtendedKeyFlag` | volatile boolean | static | Tracks 0xE0 extended key prefix for Alt/Ctrl/RShift disambiguation |
| `Keyboard[MAXKEYBOARDSCAN]` | volatile int array | global | Legacy keyboard state array (unused in active code) |
| `KeyboardQueue[KEYQMAX]` | volatile int array | global | Circular queue of raw keyboard events |
| `Keystate[MAXKEYBOARDSCAN]` | volatile int array | global | Current key down/up state per scan code |
| `Keyhead`, `Keytail` | volatile int | global | Read/write pointers for keyboard queue |
| `ticcount` | volatile int | global | Game tick counter (~35 Hz), primary game timing reference |
| `fasttics` | volatile int | global | Fast tick counter (dev only, 4× `ticcount` rate) |
| `PausePressed`, `PanicPressed` | volatile boolean | global | Key press flags for pause and panic (scroll lock) |
| `KeyboardStarted` | int | global | Initialization state flag |
| `ASCIINames[128]`, `ShiftNames[128]` | int arrays | global | Unshifted and shifted ASCII-to-scancode lookup tables |
| `timertask`, `fasttimertask` | static task\* | static | Task handles for timer scheduling |
| `TimerStarted` | static int | static | Initialization state flag |
| `pausecount` | static volatile int | static | Pause key debounce counter |
| `starttime` | static dostime_t | static | Startup time for shutdown profiling |
| `oldtimerisr`, `oldkeyboardisr` | static function pointers | static | Saved ISR vector addresses for restoration |
| `LEDs` | static int | static | Current LED bit state (caps/num/scroll) |
| `KBFlags` | static volatile int | static | Keyboard controller communication flags (ack/resend/error) |

## Key Functions / Methods

### I_TimerISR
- Signature: `void __interrupt I_TimerISR(void)`
- Purpose: Hardware timer (PIT) interrupt handler; called every ~28 ms at 35 Hz
- Inputs: None (triggered by hardware)
- Outputs/Return: None
- Side effects: Increments `ticcount`, acknowledges PIC interrupt (0x20, 0x20)
- Calls: `OUTP` macro (hardware port I/O)
- Notes: Minimal code path; runs in interrupt context. PIC acknowledge is critical.

### I_KeyboardISR
- Signature: `void __interrupt I_KeyboardISR(void)`
- Purpose: Hardware keyboard controller interrupt handler; processes raw scan codes
- Inputs: None (triggered by hardware)
- Outputs/Return: None
- Side effects: Reads scan code from port 0x60, populates `KeyboardQueue`, updates `Keystate[]`, sets `LastScan`, manages `ExtendedKeyFlag`, `PausePressed`, `PanicPressed`
- Calls: `inp`, `OUTP` (hardware I/O), `_disable()`, `_enable()` (interrupt masking)
- Notes: Handles extended keys (0xE0 prefix), pause key (0xE1 sequence), panic key (0x46), debounces with `pausecount`. Maps left shift to right shift for extended sequences. Acknowledges keyboard controller and PIC.

### I_SetTimer0
- Signature: `void I_SetTimer0(int speed)`
- Purpose: Configure system timer 0 (PIT) frequency
- Inputs: `speed` – desired frequency in Hz (validated 0 < speed < 150)
- Outputs/Return: None
- Side effects: Writes to PIT control (0x43) and data (0x40) ports; halts on invalid speed
- Calls: `OUTP`, `Error` (fatal error handler)
- Notes: Formula: `count = 1192030 / speed`. Unused in current code (task-based scheduler used instead).

### I_Delay
- Signature: `void I_Delay(int delay)`
- Purpose: Block execution for specified time (in tenths of a second)
- Inputs: `delay` – delay in tenths of a second
- Outputs/Return: None
- Side effects: Spins on `ticcount` until target reached; can early-exit on key press (`LastScan`)
- Calls: `IN_ClearKeysDown()`, spin loop
- Notes: Busy-wait loop; poor for multitasking but acceptable for DOS.

### I_StartupTimer
- Signature: `void I_StartupTimer(void)`
- Purpose: Initialize timer subsystem; set up task scheduling and synchronize with CMOS
- Inputs: None
- Outputs/Return: None
- Side effects: Reads CMOS time, creates timer tasks via `TS_ScheduleTask()`, dispatches tasks, initializes `ticcount`, sets `TimerStarted`
- Calls: `I_GetCMOSTime()`, `_dos_settime()`, `TS_ScheduleTask()`, `TS_Dispatch()`, `printf`
- Notes: Skipped if `PROFILE` macro enabled. Uses task scheduler instead of direct ISR hook (see commented-out legacy code).

### I_ShutdownTimer
- Signature: `void I_ShutdownTimer(void)`
- Purpose: Cleanup timer; terminate tasks and optionally log timing profiling
- Inputs: None
- Outputs/Return: None
- Side effects: Reads CMOS and DOS time, terminates timer tasks, calls `TS_Shutdown()`, logs time delta if `DEVELOPMENT` enabled
- Calls: `I_GetCMOSTime()`, `_dos_gettime()`, `TS_Terminate()`, `TS_Shutdown()`, `SoftError()`
- Notes: Skipped if `PROFILE` macro enabled. Includes commented legacy ISR cleanup code.

### I_GetCMOSTime
- Signature: `void I_GetCMOSTime(struct dostime_t *cmostime)`
- Purpose: Read real-time clock from CMOS
- Inputs: Pointer to output `dostime_t` structure
- Outputs/Return: Populates `cmostime->second`, `minute`, `hour` (BCD-decoded)
- Side effects: I/O port reads (0x70, 0x71)
- Calls: `OUTP`, `inp`
- Notes: Reads CMOS locations 0, 2, 4 for seconds, minutes, hours; converts BCD to decimal.

### I_SendKeyboardData
- Signature: `void I_SendKeyboardData(int val)`
- Purpose: Send command byte to keyboard controller; wait for ack/resend
- Inputs: `val` – command byte (e.g., 0xed for LED command, 0xf4 for enable)
- Outputs/Return: None
- Side effects: Disables interrupts, writes to port 0x60, spins on `KBFlags` for handshake, sets `kb_error` flag on timeout
- Calls: `_disable()`, `_enable()`, `inp`, `outp`, spin loops
- Notes: Retry logic (3 attempts); waits for kb_ack (0xfa) or kb_resend (0xfe) flags set by ISR. Critical section protected by `_disable()`.

### I_SetKeyboardLEDs
- Signature: `void I_SetKeyboardLEDs(int which, boolean val)`
- Purpose: Toggle keyboard LEDs (caps, num, scroll lock)
- Inputs: `which` – LED type (0=scroll, 1=num, 2=caps); `val` – true to enable
- Outputs/Return: None
- Side effects: Updates `LEDs` state, sends LED command sequence to keyboard controller, spins on error retry
- Calls: `_disable()`, `_enable()`, `I_SendKeyboardData()`, spin loops
- Notes: Sends 0xed (LED command), then LED bitmask, then 0xf4 (enable). Retry loop breaks after 4 attempts or successful ack.

### I_StartupKeyboard
- Signature: `void I_StartupKeyboard(void)`
- Purpose: Initialize keyboard subsystem; hook ISR and clear state
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes `LEDs`, `KBFlags`, `Keyhead`, `Keytail`, `Keystate[]`; hooks `I_KeyboardISR` at vector 0x8000|KEYBOARDINT (0x09), saves old vector
- Calls: `_dos_getvect()`, `_dos_setvect()`, `memset()`, `printf`
- Notes: Sets `KeyboardStarted` flag. Uses 0x8000 flag with `_dos_setvect` (likely a custom convention).

### I_ShutdownKeyboard
- Signature: `void I_ShutdownKeyboard(void)`
- Purpose: Cleanup keyboard; restore saved ISR vector and clear BIOS buffer
- Inputs: None
- Outputs/Return: None
- Side effects: Restores `oldkeyboardisr`, clears BIOS keyboard buffer (addresses 0x41c, 0x41a)
- Calls: `_dos_setvect()`
- Notes: Skips LED clear (commented out). Direct manipulation of BIOS data area (0x41c).

### ISR_SetTime
- Signature: `void ISR_SetTime(int settime)`
- Purpose: Set game tick counter to arbitrary value
- Inputs: `settime` – new tick count
- Outputs/Return: None
- Side effects: Writes to `ticcount`
- Calls: None
- Notes: Large block of development profiling code is compiled out (#if 0). Function is trivial: `ticcount = settime`.

### ISR_Timer
- Signature: `static void ISR_Timer(task *Task)`
- Purpose: Timer task callback; increments timer data counter
- Inputs: `Task` – task pointer (carries data payload)
- Outputs/Return: None
- Side effects: Increments `*(int*)(Task->data)` (typically points to `ticcount` or `fasttics`)
- Calls: None
- Notes: Called by task scheduler at configured rate. Large development profiling code is compiled out (#if 0).

## Control Flow Notes
**Initialization phase:** `I_StartupTimer()` and `I_StartupKeyboard()` are called during game startup, installing interrupt handlers and initializing task scheduling.

**Frame/game loop phase:** Hardware interrupts fire asynchronously:
- Timer interrupt (`I_TimerISR`) fires ~35 times per second, incrementing `ticcount`.
- Keyboard interrupt (`I_KeyboardISR`) fires on each key press/release, populating `KeyboardQueue` and updating `Keystate[]`.

Higher-level game code polls `ticcount` for frame timing and reads `KeyboardQueue`/`Keystate[]` for input.

**Shutdown phase:** `I_ShutdownTimer()` and `I_ShutdownKeyboard()` restore original ISR vectors and clean up task scheduler.

This is a **reactive** architecture: ISRs are triggered by hardware events and update shared volatile state that main code reads.

## External Dependencies
- **DOS headers:** `<dos.h>`, `<mem.h>`, `<conio.h>` – DOS interrupt/I/O/memory macros
- **task_man.h:** `TS_ScheduleTask()`, `TS_Dispatch()`, `TS_Terminate()`, `TS_Shutdown()` – task scheduler
- **rt_in.h:** `LastScan`, `IN_ClearKeysDown()` – input state (defined elsewhere)
- **rt_def.h:** Constants (`VBLCOUNTER`, `MAXKEYBOARDSCAN`, `KEYQMAX`), types (`boolean`)
- **isr.h, _isr.h:** Interrupt vector numbers (`TIMERINT`, `KEYBOARDINT`)
- **keyb.h:** Keyboard constants (scroll_lock, num_lock, caps_lock, sc_* scan codes)
- **rt_main.h, rt_util.h, profile.h, develop.h:** Utilities and profiling (indirect)
