Looking at the provided cross-reference index, I notice that **none of the SWIFT functions appear anywhere in the call graph**. This is a significant architectural insight. Let me write the enhanced analysis:

---

# rott/rt_swift.h — Enhanced Analysis

## Architectural Role

SWIFT is an optional, dynamically-integrated input subsystem for specialized 3D peripherals (e.g., SpaceBall). Unlike mandatory input systems (keyboard, mouse, joystick), SWIFT operates as a pluggable optional module: initialization succeeds only if the device and driver extensions are present, and callers must check the return status. This design allowed Apogee to ship the base game without SWIFT dependencies while enabling power-users with niche hardware to unlock additional input capabilities.

## Key Cross-References

### Incoming (who depends on this file)
- **Absence is notable:** SWIFT functions do **not appear** in the cross-reference index. This indicates either:
  - SWIFT is conditionally compiled out in the provided source snapshot
  - SWIFT calls are isolated to a single high-level input dispatcher (likely in `rt_main.c` or `rt_game.c` loop) not fully indexed
  - SWIFT was intended as a premium feature for advanced users, not core gameplay

### Outgoing (what this file depends on)
- **Includes:** `rt_playr.h` — provides typedef definitions for `SWIFT_3DStatus` and `SWIFT_StaticData` (struct definitions not visible in rt_swift.h itself; they live in rt_playr.h)
- **Memory model:** Depends on real-mode/DOS protected-mode segmented architecture (evidenced by far pointers)

## Design Patterns & Rationale

### Init/Terminate Pairing
Follows the classic resource acquisition/release pattern (RAII-adjacent). `SWIFT_Initialize()` probes for hardware and allocates resources; `SWIFT_Terminate()` is idempotent and safe to call unconditionally, enabling clean shutdown regardless of initialization success. This was common in DOS-era driver code where resource management was manual.

### Frame-Driven Polling Model
`SWIFT_Get3DStatus()` and `SWIFT_GetDynamicDeviceData()` are designed to be called **once per game frame**, mirroring the synchronous input polling model (keyboard/joystick). This matches the era's polling-based game loop rather than interrupt-driven or event-queue input.

### Tunable Haptic PWM
`SWIFT_TactileFeedback()` uses simple on/off timing parameters (`on`, `off` durations in milliseconds), effectively implementing software PWM. This is a low-bandwidth command interface suitable for real-time latency-sensitive applications.

### No Visible State
The header declares no module-level state (no static globals, no opaque handles). All state is implicit in the SWIFT device driver; the module acts as a thin wrapper around hardware queries.

## Data Flow Through This File

1. **Startup phase:**
   - `SWIFT_Initialize()` → device driver probes for hardware → returns success/fail
   - If successful, `SWIFT_GetStaticDeviceInfo()` (optional) queries immutable device metadata

2. **Per-frame polling:**
   - Game loop calls `SWIFT_Get3DStatus()` → reads 6DOF (x, y, z, pitch, roll, yaw) + button state from device
   - Game loop calls `SWIFT_GetDynamicDeviceData()` → reads device status flags (battery, error, etc.)
   - Input data flows into central input accumulator (likely merged with keyboard/mouse state in a higher layer)

3. **On-demand haptic feedback:**
   - Gameplay event (e.g., weapon fire, collision) calls `SWIFT_TactileFeedback(duration, on_ms, off_ms)`
   - Motor controller receives cycled on/off command

4. **Shutdown:**
   - `SWIFT_Terminate()` → device driver frees resources

## Learning Notes

**Idiomatic to this era & codebase:**
- Far pointers (`far *`) reflect DOS protected mode or bank-switched memory; modern engines hide this in memory allocators
- Absence from cross-reference index suggests **optional features were not deeply integrated** into core gameplay logic, preserving modularity
- The init/terminate pairing pattern predates RAII but achieves similar safety guarantees through discipline
- Polling-based input was the standard for real-time games in 1994–1995; event-driven input came later

**Connections to modern engine concepts:**
- SWIFT is a **peripheral driver abstraction layer** — analogous to modern input backends (XInput, DS4Windows API wrappers)
- The stateless design (query-response only) avoids temporal coupling; modern engines use similar patterns for hardware drivers
- 6DOF input support was cutting-edge in 1994; now standard in VR/AR frameworks (e.g., OpenXR)

## Potential Issues

1. **Silent hardware failure:** `SWIFT_Initialize()` returning 0 means "no SWIFT," but the header doesn't clarify whether partial driver failures (e.g., device present but firmware error) are also masked as 0. Calling code cannot distinguish.

2. **Unspecified feedback behavior:** `SWIFT_TactileFeedback()` contract is unclear: Is the burst duration (`d`) wallclock time or OS-dependent? What happens if called before initialization? No error codes.

3. **Type opaqueness:** `SWIFT_StaticData` and `SWIFT_3DStatus` struct contents must be read from `rt_playr.h`. If callers forget to initialize output pointers, memory corruption is silent.

4. **Far pointer portability:** The use of `far` pointers ties this interface to segmented memory architectures; porting to modern flat-memory systems required careful struct packing and casting.
