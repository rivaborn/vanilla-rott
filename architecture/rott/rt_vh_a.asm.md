# rott/rt_vh_a.asm

## File Purpose
Low-level x86 assembly module providing video hardware abstraction and joystick input routines for the ROTT engine. Handles tiled screen buffer updates to VGA memory and analog joystick position reading via resistor-capacitor timing.

## Core Responsibilities
- Implement screen tile update logic with VGA register manipulation
- Read joystick analog values using port-based resistor discharge timing
- Manage VGA write planes and graphics controller state during screen updates
- Convert raw joystick timing counts to calibrated position values

## Key Types / Data Structures
None.

## Global / File-Static State
All state is external; this file references:

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _update | DWORD array | global | Tile dirty flags (1 = needs refresh) |
| _bufferofs | DWORD | global | Base offset of source framebuffer |
| _displayofs | DWORD | global | Base offset of VGA display memory |
| _blockstarts | DWORD array | global | Offsets for each of 608 update tiles |
| _linewidth | DWORD | global | Scanline byte width |
| _Joy_x, _Joy_y | WORD | global | Output joystick position values |
| _Joy_xb, _Joy_yb | BYTE | global | Per-axis input port masks |
| _Joy_xs, _Joy_ys | BYTE | global | Per-axis right-shift calibration amounts |

## Key Functions / Methods

### VH_UpdateScreen_
- **Signature:** `void VH_UpdateScreen_()`
- **Purpose:** Selectively refresh VGA display tiles from framebuffer based on dirty flags.
- **Inputs:** None (consumes global _update, _blockstarts, _bufferofs, _displayofs, _linewidth)
- **Outputs/Return:** None (modifies VGA video memory)
- **Side effects:** 
  - Writes to VGA controller ports (SC_INDEX, GC_INDEX)
  - Copies tile data (64 bytes per tile) from buffer to display memory
  - Clears _update flag for processed tiles
- **Calls:** None (direct I/O and memory ops)
- **Notes:**
  - Loops UPDATEWIDE×UPDATEHIGH − 1 times (32×19 = 607 tiles, decrementing)
  - For flagged tiles, unrolled loop copies 16 rows × 4 bytes using _linewidth stride
  - Sets all VGA planes writable (SC_MAPMASK = 15) at start
  - Restores graphics controller mode register at end

### JoyStick_Vals_
- **Signature:** `void JoyStick_Vals_()`
- **Purpose:** Poll joystick analog position values via game port resistor-capacitor discharge timing.
- **Inputs:** _Joy_xb, _Joy_yb (axis masks), _Joy_xs, _Joy_ys (calibration shift bits)
- **Outputs/Return:** Writes _Joy_x, _Joy_y with scaled position counts
- **Side effects:**
  - Disables interrupts (CLI) for timing accuracy
  - I/O to game control port 0201h
  - Restores CPU flags (POPF)
- **Calls:** None (direct I/O and bit ops)
- **Notes:**
  - Uses MaxJoyValue (5000) loop counter as timeout guard
  - Counts discharge cycles in SI (X) and DI (Y) until capacitor voltages drop
  - Post-read applies per-axis right-shift scaling to convert counts to position
  - Timing-critical: highly sensitive to CPU speed; no modern system support expected

## Control Flow Notes
Both functions are engine entry points called from C code:
- **VH_UpdateScreen_** is invoked per-frame to refresh changed screen tiles (dirty-flag pattern)
- **JoyStick_Vals_** is polled periodically during input handling to capture joystick state

## External Dependencies
- **I/O Ports:** 0201h (game control port), 03C4h (VGA sequencer index), 03CEh (VGA graphics controller)
- **External symbols:** _bufferofs, _displayofs, _linewidth, _blockstarts, _update, _Joy_x, _Joy_y, _Joy_xb, _Joy_yb, _Joy_xs, _Joy_ys
