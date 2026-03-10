# rott/_rt_play.h

## File Purpose
Private header defining constants, macros, and forward declarations for player gameplay mechanics in Rise of the Triad. Covers player movement, weapon handling, camera/horizon control, and VR input mapping.

## Core Responsibilities
- Defines angle limits and speeds for camera tilt, horizon, and player view control
- Provides macros for weapon state transitions (`StartWeaponChange`) and player physics state queries
- Defines movement speed constants (run, base move, jump, thrust)
- Provides distance and angle calculation macros for collision and physics
- Maps VR controller buttons to gameplay actions (0–15)
- Declares forward signatures for enemy attack functions (bat, dog)

## Key Types / Data Structures
None (file uses externally-defined types: `objtype`, `MoveType`, player state).

## Global / File-Static State
None.

## Key Functions / Methods

### BatAttack
- Signature: `void BatAttack(objtype*ob);`
- Purpose: Forward declaration for bat enemy attack behavior.
- Inputs: Pointer to game object (`ob`).
- Outputs/Return: None.
- Calls: Defined elsewhere.
- Notes: Declaration only; implementation not in this file.

### T_DogLick
- Signature: `void T_DogLick(objtype *ob);`
- Purpose: Forward declaration for dog enemy lick attack behavior.
- Inputs: Pointer to game object (`ob`).
- Outputs/Return: None.
- Calls: Defined elsewhere.
- Notes: Declaration only; implementation not in this file.

---

**Notable Macros:**
- `SetPlayerHorizon(ps, hlevel)` — Clamps horizon angle within ±YZANGLELIMIT with offset.
- `StartWeaponChange` — Plays weapon-select sound, sets weapon-down ticks, updates HUD.
- `M_DIST`, `DISTANCE` — Squared distance calculations (2D).
- `W_CHANGE`, `NOMOM`, `Fix`, `DISTOK` — State and physics query helpers.

## Control Flow Notes
File is included during:
- **Input handling** — Keyboard turn amounts (`KEYBOARDNORMALTURNAMOUNT`, `TURBOTURNAMOUNT`) and VR button mapping (0–15).
- **Physics updates** — Movement scales (`BASEMOVE`, `RUNMOVE`), jump/thrust constants, momentum checks.
- **Rendering** — Horizon offset, tilt limits, camera angle constants.
- **Weapon management** — Weapon change timing, bullet hole offset positioning.

## External Dependencies
- **Include:** `watcom.h` (fixed-point math: `FixedMulShift`).
- **Undefined symbols** (defined elsewhere):
  - `FINEANGLES`, `ANG180` — Angle quantization constants.
  - `WEAPONS[]` — Weapon table (screen height, etc.).
  - `SD_PlaySoundRTP()`, `SD_SELECTWPNSND` — Sound API.
  - `player`, `ob`, `pstate` — Global/parameter game objects.
  - `SHOW_BOTTOM_STATUS_BAR()`, `DrawBarAmmo()` — UI functions.
