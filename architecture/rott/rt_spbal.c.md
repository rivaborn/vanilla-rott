# rott/rt_spbal.c

## File Purpose

Manages Spaceball 6-DOF input device integration for the ROTT game engine. Handles device initialization, real-time motion polling, button mapping, and conversion of raw Spaceball data into game control inputs (movement, aiming, weapon firing).

## Core Responsibilities

- Initialize Spaceball hardware and load device configuration from file
- Poll Spaceball for raw 6D motion (translation/rotation) and button states
- Filter and normalize raw motion data (single-axis, planar, null-zone suppression)
- Apply configurable warp/sensitivity curves to each motion axis via fixed-point math
- Map physical buttons to game actions (attack, use, map, aim, pause, turbo-fire, weapon swap)
- Convert Spaceball motion into game control buffer deltas (forward/strafe/vertical)
- Implement turbo-fire accumulation mechanic by modulating rotation input
- Shut down device cleanly on exit

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SpwRawData` | struct (from splib.h) | Raw Spaceball packet: 6D motion values (tx, ty, tz, rx, ry, rz) and button record |
| `WarpRecord` | struct (from _rt_spba.h) | Contains label, warp ranges array, and count for sensitivity curve per axis |
| `WarpRange` | struct (from _rt_spba.h) | Defines input range and sensitivity scale for motion warping |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `defaultWarps` | `WarpRange[5][1]` | static | Hardcoded sensitivity curves for Tx, Ty, Tz, Rx, Ry axes |
| `defaultRecords` | `WarpRecord[5]` | static | Warp record descriptors paired with defaultWarps |
| `WarpTx, WarpTy, WarpTz, WarpRx, WarpRy` | `WarpRecord*` | global | Pointers to active warp records (may point to config-loaded or default records) |
| `SpaceBallConfigName` | `char*` | static | Filename "spwrott.cfg" for device configuration |
| `ApogeePath` | `char*` | static | Environment variable name "APOGEECD" for config search path |
| `turbo_increment, turbo_count, turboFire` | `int, int, int` | static | Turbo-fire state: increment step, accumulator, and active flag |
| `turboFireMask...pauseMask` | `short` (8 vars) | static | Button bitmasks for each game action (default assignments A–F) |
| `masks[6]` | `short*[6]` | static | Array of pointers to button masks for conflict detection during config parsing |

## Key Functions / Methods

### ShiftTowardZero
- **Signature:** `static short ShiftTowardZero(short n, short amount)`
- **Purpose:** Suppress low-magnitude joystick noise by shifting small values toward zero (dead-zone removal).
- **Inputs:** `n` = raw input value; `amount` = dead-zone threshold.
- **Outputs/Return:** Reduced value if `|n| >= amount`, else 0.
- **Side effects:** None.
- **Calls:** `MABS` (macro).
- **Notes:** Applied to raw Spaceball axes before warping to recover low-end precision lost in null-region filtering.

### SPW_SingleAxisFilter
- **Signature:** `static void SPW_SingleAxisFilter(SpwRawData *p)`
- **Purpose:** Enforce single-axis dominance: zero all motion axes except the one with largest absolute magnitude.
- **Inputs:** `p` = pointer to SpwRawData packet (assumes tx, ty, tz, rx, ry contiguous in memory).
- **Outputs/Return:** Modifies packet in-place; no return value.
- **Side effects:** Zeroes five of six motion fields.
- **Calls:** `MABS` (macro).
- **Notes:** Activated when any axis exceeds ~450 units to prevent chaotic multi-axis coupling during aggressive device movement.

### HandleSpaceBallMotion
- **Signature:** `static void HandleSpaceBallMotion(SpwRawData *npacket, int controlbuf[])`
- **Purpose:** Convert raw Spaceball motion packet into game movement and aiming control deltas.
- **Inputs:** `npacket` = raw Spaceball data; `controlbuf[3]` = accumulator for forward/strafe (0,1) and vertical (2) motion.
- **Outputs/Return:** Modifies `controlbuf[0..2]` in-place; updates `buttonpoll[]` flags for aim/look actions.
- **Side effects:** Reads/writes global state: `player->angle`, `viewcos`, `viewsin`, `costable`, `sintable`, `buttonpoll[]`, `turboFire`, `turbo_count`, `turbo_increment`.
- **Calls:** `SPW_SingleAxisFilter`, `ShiftTowardZero`, `SbFxConfigWarp`, `FixedMul`.
- **Notes:** Applies strafe angle rotation using view-relative trigonometry. If turbo-fire active, injects turbo_increment into rotation (ry) to accumulate automatic fire. Button state gates look-up/down vs. aim-up/down behavior depending on `FL_FLEET` player flag.

### PollSpaceBall
- **Signature:** `void PollSpaceBall(void)`
- **Purpose:** Main polling entry point: acquire Spaceball input, track button transitions, dispatch motion and button actions.
- **Inputs:** None (global input device state).
- **Outputs/Return:** None (modifies global `buttonpoll[]`, `controlbuf[]`).
- **Side effects:** Calls `SpwSimpleGet()` to poll hardware; updates `buttonState`, `buttonpoll[]` array; calls `HandleSpaceBallMotion()`, `DoMap()`, `PausePressed` flag.
- **Calls:** `SpwSimpleGet`, `HandleSpaceBallMotion`, `DoMap`, `SPW_SingleAxisFilter`.
- **Notes:** Implements packet reuse (6 frames) if hardware doesn't deliver new data each frame. Tracks button state transitions. Maps button masks to `buttonpoll[]` indices. Skips motion processing if paused.

### OpenSpaceBall
- **Signature:** `void OpenSpaceBall(void)`
- **Purpose:** Initialize Spaceball hardware, run interactive test, load configuration, and set up warp records and button assignments.
- **Inputs:** None.
- **Outputs/Return:** None (sets global `SpaceBallPresent` flag and warp/button state).
- **Side effects:** Calls `SpwSimpleOpen()`, `printf()` (console output), `kbhit()` (keyboard polling), `GetPathFromEnvironment()`, `SbConfigParse()`, `SbConfigGetWarpRange()`, `SbConfigGetButtonNumber()`. Modifies global pointers `WarpTx..WarpRy` and button masks.
- **Calls:** `SpwSimpleOpen`, `SpwSimpleGet`, `GetPathFromEnvironment`, `SbConfigParse`, `SbConfigGetWarpRange`, `SbConfigGetButtonNumber`.
- **Notes:** If device opens, displays interactive motion/button test until user presses keyboard key. Loads config file from `APOGEECD/spwrott.cfg`. Falls back to hardcoded defaults if config missing or buttons not found. Button assignment parser ensures no two actions share same button (uses `masks[]` array for conflict prevention).

### CloseSpaceBall
- **Signature:** `void CloseSpaceBall(void)`
- **Purpose:** Cleanly shut down Spaceball device at engine exit.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `SpwSimpleClose()` if device was successfully opened.
- **Calls:** `SpwSimpleClose`.
- **Notes:** Guards close call with `SpaceBallPresent` flag to avoid errors if device never opened.

### GetSpaceBallButtons
- **Signature:** `unsigned GetSpaceBallButtons(void)`
- **Purpose:** Query current Spaceball button state (for external polling, e.g., menu systems).
- **Inputs:** None.
- **Outputs/Return:** Button bitmask from device (bitwise AND with `SPW_BUTTON_DOWN` mask).
- **Side effects:** Calls `SpwSimpleGet()`.
- **Calls:** `SpwSimpleGet`.
- **Notes:** Simple pass-through; no filtering or mapping applied.

## Control Flow Notes

**Initialization phase:** `OpenSpaceBall()` called once at engine startup (likely in `RT_MAIN.C` or input subsystem init). Displays interactive test, parses config, initializes warp records and button masks.

**Per-frame polling:** `PollSpaceBall()` called during input update loop (likely in `PollControls()` or equivalent input dispatcher in `RT_PLAYR.C`). Reads hardware, converts to `controlbuf[]` deltas and `buttonpoll[]` flags. Motion affects frame-to-frame player position; buttons trigger immediate actions or sustained state flags.

**Shutdown phase:** `CloseSpaceBall()` called at engine exit, releases device resources.

## External Dependencies

- **Spaceball library** (splib.h): `SpwSimpleOpen`, `SpwSimpleGet`, `SpwSimpleClose` — raw hardware I/O.
- **Spaceball config system** (_rt_spba.h, sbconfig.h): `SbConfigParse`, `SbConfigGetWarpRange`, `SbConfigGetButtonNumber` — configuration parsing and warp record retrieval.
- **Fixed-point math** (watcom.h): `FixedMul` — 16.16 fixed-point multiplication for motion scaling.
- **Game state** (rt_playr.h): `player` (global player object), `buttonpoll[]` (button polling array), `controlbuf[3]` (accumulated control delta), `PausePressed` (pause flag), angle/height constants.
- **Rendering context** (rt_draw.h): `viewcos`, `viewsin`, `costable[]`, `sintable[]` — view-relative trigonometry for strafe angle computation.
- **Utility** (rt_util.h, rt_main.h): `GetPathFromEnvironment`, `DoMap` — path resolution and map display command.
- **Math constants** (rt_def.h): `FINEANGLES`, `FL_FLEET` player flag.
- **I/O** (conio.h, io.h): `kbhit()`, `printf()` — console I/O for interactive test.
