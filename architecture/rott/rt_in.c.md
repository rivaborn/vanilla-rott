# rott/rt_in.c

## File Purpose

Input manager for ROTT that handles detection, initialization, and polling of multiple input devices (keyboard, mouse, joysticks, SpaceBall, Cyberman, Assassin). Provides normalized control reading and text input queuing for chat/menus.

## Core Responsibilities

- Initialize and detect connected input devices (keyboard, mouse, joysticks, special hardware)
- Process keyboard queue from ISR and maintain keyboard state array
- Convert raw joystick/mouse input to normalized control directions and magnitudes
- Calibrate and scale joystick analog input with threshold-based dead zones
- Provide mouse button and joystick button state to game logic
- Handle acknowledgment loops for menu waits (wait for key/button release then press)
- Queue ASCII character input for chat messages and modem text composition
- Integrate special input devices (SpaceBall, Cyberman, Assassin) via SWIFT protocol
- Shutdown and cleanup input subsystems on exit

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `KeyboardDef` | struct | Maps scan codes to button indices (button0, button1, directional) |
| `JoystickDef` | struct | Stores min/max/threshold values and scaling multipliers for calibrated joystick |
| `ControlType` | enum | Device type selector: `ctrl_Keyboard`, `ctrl_Joystick1/2`, `ctrl_Mouse` |
| `ControlInfo` / `CursorInfo` | struct | Input state: button0–3, x/y deltas, xaxis/yaxis motion, direction |
| `ModemMessage` | struct | Chat message metadata: flag `messageon`, recipient `towho`, buffer `textnum`, length |
| `Direction` | enum | 8-directional + none lookup result from motion pair |
| `Motion` | enum | Single axis: `motion_Left` (−1), `motion_None` (0), `motion_Right` (1), etc. |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `KbdDefs` | `KeyboardDef` | static | Single keyboard configuration (scan codes for buttons and directions) |
| `JoyDefs[MaxJoys]` | `JoystickDef[]` | static | Per-joystick calibration (min/max/thresh, multipliers) |
| `Controls[MAXPLAYERS]` | `ControlType[]` | static | Current input device per player |
| `IN_Started` | `boolean` | static | Initialization guard flag |
| `DirTable[]` | `Direction[]` | static | 9-element lookup table: (my+1)*3 + (mx+1) → Direction |
| `ParmStrings[]` | `char*[]` | static | Command-line param names: "nojoys", "nomouse", "spaceball", etc. |
| `btnstate[8]` | `boolean[]` | static | Debounce state for 8 buttons in ack-wait loops |
| `MousePresent` | `boolean` | global | Mouse detected and available |
| `JoysPresent[MaxJoys]` | `boolean[]` | global | Per-joystick presence flag |
| `CybermanPresent`, `SpaceBallPresent`, `AssassinPresent` | `boolean` | global | Special device flags |
| `Paused` | `boolean` | global | Game pause state |
| `LastScan` | `volatile int` | global | Last keyboard scan code from ISR |
| `LastASCII` | `char` | global | Last ASCII character |
| `Joy_x`, `Joy_y` | `word` | global | Raw joystick position values |
| `Joy_xb`, `Joy_yb`, `Joy_xs`, `Joy_ys` | `byte`, `byte`, `byte`, `byte` | global | Joystick bit/shift masks for port 0x201 |
| `LastLetter`, `LetterQueue[MAXLETTERS]` | `int`, `char[]` | global | Text input buffer for chat |
| `MSG` | `ModemMessage` | global | Active modem message composition state |
| `ScanChars[128]`, `ShiftedScanChars[128]` | `char[]` | global | Lookup tables: scan code → ASCII (shifted and unshifted) |

## Key Functions / Methods

### IN_Startup
- **Signature:** `void IN_Startup(void)`
- **Purpose:** Initialize all input devices at game startup based on command-line parameters and hardware presence
- **Inputs:** Command-line `_argv`, `_argc`; device-presence external variables
- **Outputs/Return:** Sets global presence flags; calls device-specific init functions
- **Side effects:** Modifies global state (MousePresent, JoysPresent[], CybermanPresent, etc.); may print diagnostic messages if `!quiet`
- **Calls:** `INL_StartMouse()`, `INL_StartJoy()`, `OpenSpaceBall()`, `SWIFT_Initialize()`, `SWIFT_GetDynamicDeviceData()`, `SWIFT_TactileFeedback()`, `US_CheckParm()`, `printf()`
- **Notes:** Iterates command-line arguments matching against `ParmStrings[]` to override device checks; guard `IN_Started` prevents re-initialization; sets `mouseenabled` and device-specific flags.

### IN_Shutdown
- **Signature:** `void IN_Shutdown(void)`
- **Purpose:** Clean up and disable all input devices at game shutdown
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets `IN_Started` flag; calls cleanup functions
- **Calls:** `INL_ShutJoy()` per joystick, `SWIFT_Terminate()`, `CloseSpaceBall()`
- **Notes:** Guard `IN_Started` prevents double-shutdown; note commented out `INL_ShutMouse()`

### IN_ReadControl
- **Signature:** `void IN_ReadControl(int player, ControlInfo *info)`
- **Purpose:** Sample current input for a player and fill in motion/direction/buttons
- **Inputs:** Player index, `ControlInfo` struct pointer
- **Outputs/Return:** Writes to `info->x`, `info->y`, `info->xaxis`, `info->yaxis`, `info->button0–3`, `info->dir`
- **Side effects:** Reads `Keyboard[]` array; may call joystick/mouse input functions (currently disabled via `#if 0`)
- **Calls:** `INL_GetJoyDelta()`, `INL_GetJoyButtons()`, `INL_GetMouseDelta()`, `IN_GetMouseButtons()`
- **Notes:** Currently only keyboard path is active (cases for joystick/mouse are `#if 0` disabled); direction computed via `DirTable` from motion pair; buttons 0–3 extracted from raw button bits.

### IN_UpdateKeyboard
- **Signature:** `void IN_UpdateKeyboard(void)`
- **Purpose:** Process pending keyboard queue events from ISR; update `Keyboard[]` state array
- **Inputs:** `Keyhead`, `Keytail`, `KeyboardQueue[]` from ISR
- **Outputs/Return:** None
- **Side effects:** Modifies `Keyboard[]` array (0 for up, 1 for down); advances `Keyhead` ring pointer
- **Calls:** `memset()`
- **Notes:** High bit (0x80) in queue entry marks key-up event; lower 7 bits = scan code; loop processes all pending events; carries over movement key state.

### QueueLetterInput
- **Signature:** `void QueueLetterInput(void)`
- **Purpose:** Convert keyboard events to ASCII characters; queue for text input; handle chat message composition and special key bindings
- **Inputs:** `Keyhead`, `Keytail`, `KeyboardQueue[]`, `Keyboard[]` state (Shift, Alt, etc.)
- **Outputs/Return:** Updates `LetterQueue[]`, `LastLetter`; modifies `MSG` structure; calls message handlers
- **Side effects:** Queues printable characters; modifies `Keyboard[]`, `KeyboardQueue[]`; calls `UpdateModemMessage()`, `FinishModemMessage()`, etc. for active messages; emits debug/macro messages
- **Calls:** `FinishModemMessage()`, `UpdateModemMessage()`, `ModemMessageDeleteChar()`, `AddMessage()`, character lookup from `ScanChars[]` / `ShiftedScanChars[]`
- **Notes:** Handles in-game chat menu (recipient selection via 'A'/'T'/0–9), Alt+1–0 macro shortcuts, backspace/enter/escape in message mode; checks `MSG.messageon`, `MSG.inmenu` to gate behavior.

### IN_GetJoyAbs
- **Signature:** `void IN_GetJoyAbs(word joy, word *xp, word *yp)`
- **Purpose:** Read absolute joystick position from hardware
- **Inputs:** Joystick index (0 or 1); output pointer arguments
- **Outputs/Return:** Writes `*xp`, `*yp` with raw joystick position
- **Side effects:** Sets static variables `Joy_x`, `Joy_y`, `Joy_xs`, `Joy_ys`, `Joy_xb`, `Joy_yb`; calls hardware read
- **Calls:** `JoyStick_Vals()` (hardware-specific, defined elsewhere)
- **Notes:** Sets shift/mask values before hardware call; second joystick uses different bit positions.

### INL_GetJoyDelta
- **Signature:** `void INL_GetJoyDelta(word joy, int *dx, int *dy)`
- **Purpose:** Convert absolute joystick position to relative motion (±127 range) using calibrated thresholds and scaling
- **Inputs:** Joystick index; output pointers for delta
- **Outputs/Return:** Writes `*dx`, `*dy` with scaled motion in range [−127, 127]
- **Side effects:** Reads global `lasttime` (not used currently)
- **Calls:** `IN_GetJoyAbs()`
- **Notes:** Applies `JoystickDef` thresholds and scaling multipliers; clamps to max values; separate logic for low/high ranges per axis.

### IN_WaitForKey
- **Signature:** `ScanCode IN_WaitForKey(void)`
- **Purpose:** Busy-wait for next keyboard scan code; clear and return it
- **Inputs:** None
- **Outputs/Return:** Scan code value
- **Side effects:** Blocks until `LastScan != 0`; zeros `LastScan` on return
- **Calls:** None
- **Notes:** Polling loop; used by menu code.

### IN_Ack
- **Signature:** `void IN_Ack(void)`
- **Purpose:** Waits for a button or key press, with debounce: must be released first, then pressed
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `IN_StartAck()` then spins on `!IN_CheckAck()`
- **Calls:** `IN_StartAck()`, `IN_CheckAck()` in busy loop
- **Notes:** Used by menus; `btnstate[8]` tracks initial button state; a press is recognized only after state change from pressed→released.

### IN_UserInput
- **Signature:** `boolean IN_UserInput(long delay)`
- **Purpose:** Wait for input or timeout (whichever comes first)
- **Inputs:** Delay in ticks
- **Outputs/Return:** `true` if input detected, `false` if timeout
- **Side effects:** Reads `ticcount` global
- **Calls:** `IN_StartAck()`, `IN_CheckAck()`
- **Notes:** Non-blocking variant of `IN_Ack()`; computes elapsed time via `ticcount`.

### Mouse/Joystick calibration helpers
- **INL_StartMouse()** / **INL_StartJoy()** / **IN_SetupJoy()** / **INL_SetJoyScale()**: Hardware detection and scaling setup via INT 33h (mouse) and direct port reads (joystick). See individual definitions.

---

## Control Flow Notes

**Initialization phase** (`IN_Startup`):
- Called once at game startup
- Detects mice, joysticks, special devices based on command-line flags
- Stores device presence in global flags; configures joystick scaling

**Per-frame input phase**:
- ISR fills `KeyboardQueue` on keyboard interrupt (via `isr.h`)
- `IN_UpdateKeyboard()` drains queue, updates `Keyboard[]` state array
- `IN_ReadControl(player, info)` reads current `Keyboard[]` and optional joystick/mouse, fills `ControlInfo`
- Game applies `ControlInfo` to player movement

**Text input phase** (during chat/menus):
- `QueueLetterInput()` converts keyboard events to ASCII and queues for message composition
- Handles special keys (Enter, Escape, Backspace) and macro shortcuts
- Calls modem message API to append/finish messages

**Menu acknowledgment** (IN_Ack, IN_UserInput):
- `IN_StartAck()` captures initial button state
- Spin-loop `IN_CheckAck()` until new press detected
- Used to prevent accidental re-triggering of menu items

**Shutdown phase** (`IN_Shutdown`):
- Disables all devices and clears flags

## External Dependencies

**Hardware/OS interfaces:**
- `<dos.h>`, `<i86.h>`: DOS real-mode interrupt and port I/O
- `int386()`: Watcom C function to invoke real-mode INT (mouse INT 33h, etc.)
- `inp()`: Port I/O read (joystick at port 0x201, keyboard at port 0x60)
- ISR keyboard queue (`Keyhead`, `Keytail`, `KeyboardQueue[]`) defined in `isr.h`

**Game subsystems (defined elsewhere):**
- `OpenSpaceBall()`, `GetSpaceBallButtons()`, `CloseSpaceBall()` — SpaceBall device (`rt_spball.h`)
- `SWIFT_Initialize()`, `SWIFT_GetDynamicDeviceData()`, `SWIFT_TactileFeedback()`, `SWIFT_Terminate()` — Cyberman/Assassin 3D controller (`rt_swift.h`)
- `JoyStick_Vals()` — Joystick port read (implementation in `_rt_in.c` or asm)
- `FinishModemMessage()`, `UpdateModemMessage()`, `ModemMessageDeleteChar()`, `AddMessage()` — Modem chat subsystem (`rt_com.h`, `rt_net.h`)
- `US_CheckParm()` — Utility string param check (`rt_util.h`)
- Global game state: `gamestate`, `consoleplayer`, `numplayers`, `CommbatMacros[]`, `Messages[]` (from `rt_playr.h`, `rt_msg.h`, etc.)
- Configuration globals: `mouseenabled`, `spaceballenabled`, `cybermanenabled`, `assassinenabled`, `quiet` (from various headers)
