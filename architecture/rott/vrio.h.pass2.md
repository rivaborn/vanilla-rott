# rott/vrio.h — Enhanced Analysis

## Architectural Role
This file documents ROTT's interface to Virtual Reality hardware, specifically the VR input device abstraction layer. It bridges the gap between raw VR controller hardware (accessed via DOS interrupts INT 0x33) and the game's input event system. The file is part of a larger input subsystem that abstracts hardware differences, sitting alongside mouse/keyboard input handling.

## Key Cross-References
### Incoming (who depends on this file)
The cross-reference index provided does not show any explicit function calls to VR input routines, suggesting:
- VR input handling is likely implemented in an interrupt handler (not visible in the symbol map)
- VR controller support may have been optional or debug-only in the released build
- Integration with the main input loop likely happens in `rt_playr.c` (player control) or a dedicated input handler not shown in cross-references

### Outgoing (what this file depends on)
This is a pure documentation header—no runtime dependencies. The file documents two DOS interrupt handlers (0x33) that are presumably implemented elsewhere in the engine or by third-party VR hardware drivers.

## Design Patterns & Rationale
**Interrupt-Based Polling**: The GetVRInput design uses the DOS interrupt convention, which was standard for hardware abstraction in the early 1990s. This allows hardware vendors to drop in their own INT 0x33 handler.

**Register-Based Parameter Passing**: Uses x86 real-mode calling conventions:
- Input parameters (player state) passed in BX, CX
- Output parameters (button state, mouse movement) returned in BX, CX, DX

This is extremely efficient for low-latency input, avoiding memory allocations.

**Haptic Feedback Symmetry**: VRFeedback (0x31) mirrors GetVRInput (0x30) in register usage, suggesting a paired request-response protocol. The weapon type encoding (gun=0, missile=1) hints at expected gameplay feedback patterns.

## Data Flow Through This File
```
Game loop → GetVRInput interrupt (INT 0x33 AX=0x30)
  Inputs: current player angle + tilt angle
  Outputs: 16-bit button bitmask + mouse X/Y mickeys
         → mapped to player movement/actions in rt_playr.c

Gameplay event (weapon fire) → VRFeedback interrupt (INT 0x33 AX=0x31)
  Inputs: feedback enable/disable flag + weapon type
         → controller vibrates/pulses on hardware
```

## Learning Notes
**DOS-Era Hardware Integration**: This exemplifies 1990s real-time input abstraction. Modern engines use event-driven callbacks; ROTT polls via interrupts for deterministic timing.

**Angle Representation**: The 0..2047 range (not 0..360 or 0..256) represents fixed-point angles optimized for lookup tables (power of 2). Vertical angles wrap asymmetrically (0–171 up, 1876–2047 down), suggesting that 2047 is the angular "wrap point."

**16-Button Limit**: The button layout (bits 0–15) is dense: movement (4 buttons), actions (4 buttons), menu (4 buttons), weapon select (2 buttons), recording (1 button). This was typical for niche VR peripherals of that era.

**Sparse Cross-Reference**: The absence of VR handlers in the function map suggests either: (a) this was an optional feature compiled conditionally, or (b) VR support was complete but isolated in a separate module not analyzed in the cross-reference pass.

## Potential Issues
- **No Constants Defined**: Button names (VR_RUNBUTTON, etc.) are documented but not defined as `#define` constants in this header, likely forcing uses elsewhere to hardcode bit positions or rely on convention.
- **No Error Handling**: The interrupt protocol specifies no error return codes, making it impossible to detect VR device failure at runtime.
- **Limited Feedback**: Only 2 weapon types (gun, missile) are supported for haptic feedback; a generic damage/impact feedback might be more flexible.
- **Unbuffered Input**: No mention of input buffering or timestamping; rapid polling could miss inputs if the game loop stalls.
