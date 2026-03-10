# rott/rt_spbal.c — Enhanced Analysis

## Architectural Role

This file implements the hardware abstraction layer for Spaceball 6-DOF input devices within ROTT's input subsystem. It bridges the low-level Spaceball library (vendor-supplied) and the high-level game action system, translating raw 6-axis motion vectors and button states into normalized game control signals (`controlbuf[]` and `buttonpoll[]`). It operates as a specialized input device handler alongside standard keyboard/mouse polling, executing once per frame during the main control loop to accumulate directional and action impulses for the player object.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_playr.c / rt_main.c** (inferred): Calls `PollSpaceBall()` during per-frame input update phase, expects it to populate `controlbuf[0..2]` (forward/strafe/vertical deltas) and `buttonpoll[bt_*]` flags
- **Main game loop initialization**: Calls `OpenSpaceBall()` at startup; calls `CloseSpaceBall()` at shutdown
- **Menu/UI subsystems** (inferred): May call `GetSpaceBallButtons()` to query raw button state for interactive menu navigation

### Outgoing (what this file depends on)
- **Player state (rt_playr.h)**: Reads `player->angle` (for view-relative strafe), `player->flags & FL_FLEET` (for flight mode gating), writes indirectly via `buttonpoll[]` and `controlbuf[]`
- **Game action interface**: Writes to `buttonpoll[bt_attack], buttonpoll[bt_use], ...` (bit-indexed action flags)
- **Rendering state (rt_draw.h)**: Reads `viewcos`, `viewsin`, `costable[]`, `sintable[]` for trigonometric strafe-angle calculation
- **Map subsystem (rt_map.c)**: Calls `DoMap(player->tilex, player->tiley)` when map button pressed
- **Pause state (rt_main.h or inferred)**: Writes to global `PausePressed` flag; reads it to gate motion processing
- **Spaceball hardware library (splib.h)**: Calls `SpwSimpleOpen()`, `SpwSimpleGet()`, `SpwSimpleClose()` for raw I/O
- **Spaceball config system (sbconfig.h)**: Calls `SbConfigParse()`, `SbConfigGetWarpRange()`, `SbConfigGetButtonNumber()` to load sensitivity curves and button mappings
- **Utility layer (rt_util.h)**: Calls `GetPathFromEnvironment()` for config file path resolution
- **Fixed-point math (watcom.h)**: Uses `FixedMul()` for scaled motion multiplication
- **Math tables**: Uses global `costable[]`, `sintable[]` (populated by rendering system)

## Design Patterns & Rationale

**Sensitivity Curve Injection (Warp Records):** Rather than hard-coding axis responses, the file uses configurable lookup table warping (`WarpRecord`, `WarpRange`) supplied by the Spaceball config system. This decouples device tuning from code—users can tweak `spwrott.cfg` without recompilation. Defaults (`defaultWarps`, `defaultRecords`) ensure the device works out-of-the-box if config is missing.

**Button Masking & Conflict Prevention:** Eight actions (turbo-fire, attack, use, map, swap-weapon, single-axis filter, planar filter, aim, pause) can be independently assigned to buttons A–F. The `masks[]` array acts as a conflict detector during config parsing, preventing two actions from binding to the same button. This is defensive against malformed config files.

**Packet Reuse Strategy:** `PollSpaceBall()` caches the last hardware packet and replays it up to 6 times if the device doesn't deliver fresh data each frame. This smooths jerky input and ensures the device feels responsive even if polled only every N hardware cycles.

**Motion Filtering Modes:**
- **Single-axis dominance** (`SPW_SingleAxisFilter`): Activated when any axis exceeds 450 units, forces one-axis-at-a-time to prevent chaotic multi-axis coupling during aggressive movement.
- **Planar filtering** (config-bindable): Suppresses vertical (Y) and rotation (RZ) when button held, constraining motion to horizontal translation/rotation only.
- **Dead-zone suppression** (`ShiftTowardZero`): Recovers low-end precision by shifting small values toward zero before warping.

**Turbo-Fire Accumulation:** Rather than simple toggling, turbo-fire uses a ping-pong counter (`turbo_count`, `turbo_increment`) that oscillates between 0 and `TURBO_LIMIT`, injecting the counter delta into rotation (ry) to produce sustained but modulated firing pulses.

## Data Flow Through This File

1. **Initialization** (`OpenSpaceBall`):
   - Attempt hardware open via `SpwSimpleOpen()`
   - If successful, display interactive test loop until keypress
   - Parse config file (`spwrott.cfg`) from `APOGEECD` environment path
   - Retrieve warp records (sensitivity curves) for each axis; fall back to hardcoded defaults
   - Retrieve button-to-action mappings; initialize button masks with defaults (A=turbo, B=attack, etc.)
   - Set `SpaceBallPresent` global flag

2. **Per-Frame Polling** (`PollSpaceBall`):
   - Call `SpwSimpleGet()` to fetch fresh hardware packet (6-axis motion + 6 buttons)
   - If no fresh packet, reuse cached packet (up to 6 frames); if cache exhausted and buttons idle, return early
   - Compare button state against previous frame; track transitions
   - Dispatch button actions: set `buttonpoll[]` flags, call `DoMap()` for map button, set `PausePressed` for pause button
   - If not paused, call `HandleSpaceBallMotion()` to process motion

3. **Motion Processing** (`HandleSpaceBallMotion`):
   - If aggressive movement detected (any axis > 450), apply single-axis filter
   - Apply dead-zone suppression (`ShiftTowardZero`) to recover low-end precision
   - Warp each axis via `SbFxConfigWarp()` to apply sensitivity curves
   - If turbo-fire active, inject turbo counter into ry
   - Compute strafe angle relative to view direction
   - Apply view-relative trigonometry to convert device-frame motion to world-frame deltas
   - Accumulate into `controlbuf[0]` (forward/strafe), `controlbuf[1]` (strafe), `controlbuf[2]` (vertical/rotation)
   - If player has flight ability, map ty to lookup/lookdown; otherwise map rx (pitch) to aim or look based on aimMask button

4. **Shutdown** (`CloseSpaceBall`):
   - Call `SpwSimpleClose()` to release device

## Learning Notes

**Spaceball-Specific Era Pattern:** This 1995 code reflects the niche adoption of 6-DOF input in PC gaming. Modern engines either abstract input devices through a standardized input event system (not axis-based polling) or use middleware that handles device virtualization transparently. ROTT's direct polling model with device-specific filtering logic is tightly coupled to Spaceball hardware.

**Fixed-Point Math Everywhere:** Sensitivity curves and motion multiplication use `FixedMul()` (16.16 fixed-point) rather than floats. This was essential on 1995 x86 hardware where floating-point was significantly slower. Modern engines default to floats and only use fixed-point for networking or determinism.

**Trig Table Reuse:** The code reuses global sine/cosine tables (`costable[]`, `sintable[]`) populated by the rendering system for screen projection. This is a space optimization—avoids duplicating lookup tables—but creates hidden coupling. A modern engine would have a dedicated math library.

**Manual State Sequencing:** No state machine; the code uses nested conditionals and static variables (`buttonState`, `lastPacket`, `reusePacketNTimes`) to track per-frame deltas. A modern input system would use an event queue or observer pattern.

**Angle Normalization:** Strafe angle computed as `(player->angle - FINEANGLES/4) & (FINEANGLES-1)` uses bitwise masking for modulo wrap-around, assuming `FINEANGLES` is a power of 2. This is fast and idiomatic to the era but opaque to modern developers.

## Potential Issues

1. **Contiguity Assumption (rt_spbal.c:128):** `SPW_SingleAxisFilter` assumes the six motion fields in `SpwRawData` are contiguous in memory (`&(p->tx), ... , array_length = 6`). If the struct is padded or fields reordered, this pointer walk will corrupt data. No assertion guards this.

2. **Dead-Zone Thresholds Hardcoded:** Values like 30, 50, 60 for `ShiftTowardZero` are hardcoded per-axis. These should ideally be configurable or derived from device specs; currently they're magic constants that may not suit all Spaceball variants.

3. **Button Conflict Detection Incomplete:** The `masks[]` conflict prevention in `OpenSpaceBall` only checks at config parse time. If a button is unmapped in config, its slot in `masks[]` becomes a dangling pointer—later config entries could still assign to it via `masks[btn]`, creating subtle double-binding bugs.

4. **No Error Recovery in Motion Warp:** If `SbFxConfigWarp()` fails or returns invalid data, the code does not validate or clamp the result before accumulating into `controlbuf[]`. Corruption in the warp function could spike movement.

5. **Turbo-Fire Ping-Pong Unbounded:** The `turbo_count` oscillator has no hysteresis or damping. If the increment is very small, the counter could stall near the limit, producing irregular fire bursts.
