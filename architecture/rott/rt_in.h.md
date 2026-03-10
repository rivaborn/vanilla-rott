# rott/rt_in.h

## File Purpose
Public input system header for RT_IN.C. Defines interfaces for polling keyboard, mouse, joystick, and networking/modem input, translating hardware signals into unified control structures for player movement and actions.

## Core Responsibilities
- Define input state data structures (CursorInfo, ControlInfo, keyboard/joystick definitions)
- Declare device initialization/shutdown for keyboard, mouse, joystick
- Provide control polling and translation (scan code → motion/direction)
- Support multi-input modes (keyboard, mouse, joystick, network play)
- Manage input queue state (letter queue for text entry)
- Define modem message structures for networked play

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Motion | enum | Directional input: left/right/up/down/none |
| Direction | enum | 8-point compass directions + none |
| ScanCode | typedef | Keyboard scan code (byte) |
| ControlType | enum | Input device type: keyboard (2 variants), joystick (2 variants), mouse |
| CursorInfo | struct | Input state: 4 buttons, x/y position, xaxis/yaxis motion, direction |
| ControlInfo | typedef | Alias for CursorInfo |
| KeyboardDef | struct | Scan codes mapping for actions (button0/1, 8 directional keys) |
| JoystickDef | struct | Joystick calibration: min/max/threshold X/Y + multipliers |
| ModemMessage | struct | Modem/network message state: on/directed flags, text number, length |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MousePresent | boolean | extern | Mouse detected |
| JoysPresent[] | boolean array | extern | Per-joystick presence flags |
| JoyPadPresent | boolean | extern | Game pad detected |
| SpaceBallPresent, CybermanPresent, AssassinPresent | boolean | extern | Special input device flags |
| Paused | boolean | extern | Game pause state |
| LastScan | volatile int | extern | Last keyboard scan code |
| LastASCII | char | extern | Last ASCII character |
| KbdDefs | KeyboardDef | extern | Current keyboard mapping |
| JoyDefs[] | JoystickDef array | extern | Joystick calibrations (per joystick) |
| Controls[] | ControlType array | extern | Current control device per player |
| Joy_x, Joy_y | word | extern | Joystick raw position |
| Joy_xb, Joy_yb, Joy_xs, Joy_ys | byte | extern | Joystick button/state bytes |
| LetterQueue[] | char array (MAXLETTERS=32) | extern | Text input queue |
| LastLetter | int | extern | Queue write position |
| MSG | ModemMessage | extern | Current modem message state |
| ScanChars[] | char array (128) | extern | Scan code to ASCII mapping |
| mouseadjustment, threshold | int | extern | Input sensitivity parameters |

## Key Functions / Methods

### IN_Startup
- **Signature:** `void IN_Startup(void)`
- **Purpose:** Initialize input system; detect and configure all input devices
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets up keyboard interrupt handlers, detects/initializes mouse and joystick hardware
- **Calls:** (implementation in RT_IN.C)
- **Notes:** Must be called before any input polling; detects connected hardware via INL_StartMouse/INL_StartJoy

### IN_Shutdown
- **Signature:** `void IN_Shutdown(void)`
- **Purpose:** Shut down input system and disable hardware
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Restores keyboard handler, disables mouse/joystick
- **Calls:** (INL_ShutMouse, INL_ShutJoy in implementation)
- **Notes:** Inverse of IN_Startup

### IN_ReadControl
- **Signature:** `void IN_ReadControl(int player, ControlInfo *info)`
- **Purpose:** Poll current control state for a player
- **Inputs:** `player` (0 to MAXPLAYERS-1), `info` (output buffer)
- **Outputs/Return:** Populates `info` with button states, axes, computed motion and direction
- **Side effects:** Reads hardware state, updates motion/direction from raw axis values
- **Calls:** Input device-specific polling (joystick, mouse, or keyboard)
- **Notes:** Called per frame; abstracts underlying device type (ControlType) from caller

### IN_SetControlType
- **Signature:** `void IN_SetControlType(int player, ControlType type)`
- **Purpose:** Assign input device for a player
- **Inputs:** `player` index, `type` (ctrl_Keyboard/Joystick/Mouse + variant)
- **Outputs/Return:** None
- **Side effects:** Updates Controls[] array; subsequent IN_ReadControl calls use new device
- **Calls:** (none visible in header)
- **Notes:** Allows dynamic device switching

### IN_WaitForKey
- **Signature:** `ScanCode IN_WaitForKey(void)`
- **Purpose:** Blocking keyboard input; wait for next key press
- **Inputs:** None
- **Outputs/Return:** Scan code of pressed key
- **Side effects:** Blocks until key is released and pressed again
- **Calls:** (none visible in header)
- **Notes:** Used for menu/prompt input

### IN_WaitForASCII
- **Signature:** `char IN_WaitForASCII(void)`
- **Purpose:** Blocking keyboard input; wait for next key as ASCII character
- **Inputs:** None
- **Outputs/Return:** ASCII character
- **Side effects:** Blocks until key released/pressed
- **Calls:** (none visible in header)
- **Notes:** Translates scan code via ScanChars table

### IN_UserInput
- **Signature:** `boolean IN_UserInput(long delay)`
- **Purpose:** Check for user activity with timeout
- **Inputs:** `delay` (milliseconds or tics)
- **Outputs/Return:** true if user provided input, false if timeout
- **Side effects:** May consume input
- **Calls:** (none visible in header)
- **Notes:** Used for menu timeouts, attract mode

### IN_GetJoyAbs
- **Signature:** `void IN_GetJoyAbs(word joy, word *xp, word *yp)`
- **Purpose:** Get absolute joystick position
- **Inputs:** `joy` (joystick index), `xp`, `yp` (output pointers)
- **Outputs/Return:** Populates x, y with raw joystick coordinates
- **Side effects:** None
- **Calls:** (none visible in header)
- **Notes:** Raw values before scaling; calibration stored in JoyDefs[]

### QueueLetterInput
- **Signature:** `void QueueLetterInput(void)`
- **Purpose:** Queue keyboard input for text entry (e.g., player names)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates LetterQueue[], LastLetter
- **Calls:** (none visible in header)
- **Notes:** Handles backspace and character filtering; limited to MAXLETTERS

### INL_GetMouseDelta, IN_GetMouseButtons, INL_GetJoyDelta, INL_GetJoyButtons, INL_StartJoy, INL_ShutJoy
- Low-level hardware access; see Notes section below.

## Control Flow Notes
**Initialization/Frame/Shutdown cycle:**
- **Init:** IN_Startup() detects hardware and sets up interrupt handlers
- **Frame:** IN_ReadControl(player, info) polls the active device (Keyboard/Joystick/Mouse) and populates ControlInfo with normalized motion and direction
- **Blocking Input:** IN_WaitForKey() and IN_WaitForASCII() block until key input
- **Shutdown:** IN_Shutdown() restores keyboard and disables hardware

**Device Configuration:**
- IN_SetControlType() selects device per player
- IN_SetupJoy() and INL_SetJoyScale() calibrate joystick thresholds
- IN_Default() initializes control layout

**Text Entry:**
- QueueLetterInput() fills LetterQueue for player name entry; checked via LastLetter

**Networking/Modem:**
- ModemMessage (MSG) and related globals support modem-play message handling (directed to player, ridicule flag, etc.)

## External Dependencies
- **develop.h:** Build configuration flags (SHAREWARE, SUPERROTT, WEAPONCHEAT, etc.)
- **rottnet.h:** Networking constants (MAXPLAYERS, MAXNETNODES) and rottcom_t structure for driver communication
- **Implicit:** Assumes byte, word, boolean, int types and interrupt-driven keyboard handler (LastScan volatile)
