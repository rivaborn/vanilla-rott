# rott/rt_swift.c — Enhanced Analysis

## Architectural Role

This file implements an **optional hardware abstraction layer (HAL) for the Cyberman 3D input device**, a specialized VR controller supported on DOS-era game systems. It bridges the engine's protected-mode code with the real-mode SWIFT extensions (via DPMI INT 0x31 calls) to safely query device state and generate haptic feedback. SWIFT initialization is opt-in during engine startup; if the device is absent or unsupported, the game proceeds without 3D input. The module follows the era-standard pattern of wrapping real-mode interrupts in protected-mode abstractions.

## Key Cross-References

### Incoming (who depends on this file)
- **Input system** (likely `rt_playr.c` or similar): Calls `SWIFT_Get3DStatus()` each frame to poll 3D device state (position, buttons, orientation). The initialization/termination is called during engine startup/shutdown (likely from main game init).
- **Feedback/interaction code**: Calls `SWIFT_TactileFeedback()` on demand to generate haptic pulses (e.g., on weapon fire or collision).
- **Device detection**: Engine startup probes `SWIFT_Initialize()` and `SWIFT_GetAttachedDevice()` to determine if Cyberman is available; if not, falls back to keyboard/mouse input.

### Outgoing (what this file depends on)
- **DPMI runtime** (`int386()`, `int386x()`, `segread()`, `_dos_getvect()`): Mediates protected/real-mode transitions via INT 0x31.
- **Mouse driver** (INT 0x33 + SWIFT extensions): Accessed only through DPMI; no direct real-mode calls.
- **DOS extender memory model**: Uses `allocDOS()` and `freeDOS()` to manage DOS real-mode buffers in the 1 MB physical address space.
- **Engine debug logging**: `SoftError()` (from `rt_def.h`) for conditional debug output; compiled out in release builds.
- **C runtime**: `memset()` for zeroing interrupt info structures.

## Design Patterns & Rationale

1. **Protected/Real-Mode Bridging via DPMI**
   - Avoids direct real-mode calls from protected code; instead uses DPMI INT 0x31 to simulate interrupts.
   - `MouseInt()` encapsulates the DPMI plumbing; all device I/O goes through this single function.
   - Provides memory safety: DOS buffer (`pdosmem`) is allocated and freed explicitly, not scattered through device code.

2. **Resource Initialization/Cleanup Pattern**
   - `SWIFT_Initialize()` and `SWIFT_Terminate()` are paired; mirrors engine conventions for subsystem lifecycle.
   - Defensive: `SWIFT_Initialize()` checks prerequisites (mouse driver present, device attached) before allocating resources.
   - Safe to call `SWIFT_Terminate()` even if init never ran or failed (checks `fActive` and `pdosmem` guards).

3. **Lazy Device State Polling**
   - No interrupt handler; instead, `SWIFT_Get3DStatus()` is polled per-frame, matching the engine's synchronous input model.
   - Caller is responsible for invoking the query; no background threads or callbacks (consistent with 1994-era single-threaded DOS games).

4. **Real-Mode Segment/Selector Duality**
   - DOS buffer is allocated with both a **real-mode segment** (`segment`) and a **protected-mode selector** (`selector`).
   - Real-mode segment passed to device driver (in RMI.es); protected-mode pointer (`pdosmem`) used to read/write buffer from protected code.
   - This duality is specific to DPMI and the Rational DOS extender memory model.

## Data Flow Through This File

```
[Engine Startup]
    └─→ SWIFT_Initialize()
        ├─→ Check mouse driver (INT 0x33 vector)
        ├─→ Allocate DOS buffer via DPMI 0x0100
        ├─→ SWIFT_GetStaticDeviceInfo() 
        │   └─→ MouseInt() (DPMI INT 0x31 → INT 0x33 0x53C1)
        │       └─→ Reads device type, version into DOS buffer
        └─→ Set fActive, nAttached

[Per-Frame Input Loop]
    └─→ SWIFT_Get3DStatus()
        └─→ MouseInt() (DPMI → INT 0x33 0x5301)
            └─→ Writes 3D position/buttons to DOS buffer
            └─→ Caller reads from `*pstat` (copied from DOS buffer)

[Haptic Feedback (On Demand)]
    └─→ SWIFT_TactileFeedback(duration, on, off)
        └─→ MouseInt() (DPMI → INT 0x33 0x5330)
            └─→ Motor pulses sent to Cyberman

[Engine Shutdown]
    └─→ SWIFT_Terminate()
        └─→ freeDOS() (DPMI 0x0101)
            └─→ Release DOS buffer
        └─→ Set fActive = 0
```

## Learning Notes

1. **Era-Specific Hardware**: Cyberman was a niche 3D input device (marketed by Victormaxx, ~1993–1995). Supporting it shows ROTT's effort to leverage exotic peripherals; this subsystem would be compiled out in most modern builds.

2. **DPMI Programming**: This file demonstrates standard DOS-extender patterns:
   - Interrupt info structure (`rminfo`) passed to DPMI INT 0x31 function 0x0300 (Simulate Real-Mode Interrupt).
   - Segment/offset → protected-mode pointer conversion (`pprot = (void far *)((unsigned)segment << 4)`).
   - DOS memory allocation via DPMI function 0x0100/0x0101 (no direct `int 21h` calls).

3. **Callback-Free Input**: Unlike modern systems (with interrupt handlers or event queues), this uses synchronous polling. Caller must invoke `SWIFT_Get3DStatus()` each frame; no background thread.

4. **Quantization & Hardware Constraints**: `SWIFT_TactileFeedback()` quantizes parameters (duration /40 ms, on/off /5 ms), reflecting hardware limitations of the Cyberman motor.

5. **Debug Logging Conditionals**: Extensive `#ifdef DBUG` blocks (note typo: should be `DEBUG`); release builds have zero overhead for this subsystem if not in use.

## Potential Issues

1. **Typo in Debug Macro**: Line 51 has `#ifdef DEGUB` (should be `DBUG`), so that specific error message will never print even in debug builds. Other occurrences use `DBUG` correctly.

2. **Bit-Shift Precedence in `SWIFT_TactileFeedback()`** (line 304):
   ```c
   RMI.bx = (on / 5) << 8 + (off / 5);
   ```
   Due to operator precedence, this parses as `(on / 5) << (8 + (off / 5))`, not `((on / 5) << 8) + (off / 5)`. Should use parentheses: `((on / 5) << 8) | (off / 5)`.

3. **No Error Checking in `freeDOS()`**: Unlike `allocDOS()`, `freeDOS()` does not verify the DPMI call succeeded (no `cflag` check). If selector is invalid, failure is silent.

4. **Unguarded DOS Buffer Access**: Functions like `SWIFT_GetStaticDeviceInfo()` assume `pdosmem` is valid without checking. If `SWIFT_Initialize()` failed (e.g., `allocDOS()` returned NULL), subsequent calls would crash dereferencing `pdosmem`.

5. **No Bounds Checking**: `SWIFT_TactileFeedback()` accepts arbitrary `d`, `on`, `off` values; invalid parameters are silently quantized, not rejected.

6. **Far Pointer Assumptions**: The `far` keyword and protected/real segmentation are specific to 16-bit DOS extenders. This code is not portable to 32-bit flat memory models (modern Windows/Linux).
