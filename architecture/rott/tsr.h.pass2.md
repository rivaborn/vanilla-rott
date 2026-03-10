# rott/tsr.h — Enhanced Analysis

## Architectural Role
This file defines the **low-level hardware abstraction layer** for DOS Terminate-and-Stay-Resident (TSR) device drivers, specifically supporting force-feedback and spatial input devices (like 3Dfx-era SpaceBall controllers). It bridges the game engine (via high-level subsystems like `rt_spbal.c`) to DOS interrupt-driven hardware drivers using x86 real-mode register calling conventions. The protocol enables device enumeration, capability queries, and real-time force/button data polling.

## Key Cross-References

### Incoming (who depends on this file)
- **`rott/rt_spbal.c / rt_spbal.h`** (SpaceBall support module) — likely primary consumer, uses these packet structures and command codes to query and read 6-DOF input devices
- **Any input handling subsystem** that needs to support optional 3D input devices beyond keyboard/mouse

### Outgoing (what this file depends on)
- **DOS/x86 interrupt subsystem** (implicit) — callers must issue INT calls with populated AX/DX/ES:BX registers
- **`SPWERR_TSR` error code base** (defined elsewhere, likely `rt_cfg.h` or similar) — error code macros inherit this prefix

## Design Patterns & Rationale

**Union-based variant messaging**: `TsrPacket` is a discriminated union. Callers populate `command.data` with magic value, then the driver interprets the union member matching the command type. This saves memory and provides type-safe (at protocol level) polymorphic communication — characteristic of fixed-size packet protocols.

**Capability-flag dispatch**: `TSR_NEED_PACKET` (0x8000) high bit indicates commands requiring a payload buffer. This keeps the command namespace compact and makes validation trivial (single bit test).

**Register-based calling convention**: Follows DOS INT calling standards — caller responsibility to set AX/DX/BX registers before INT, driver returns result in AX. This is the only practical communication method in real-mode DOS.

**Conditional private structures**: `#ifdef PRIVATE_STRUCTS` gates driver configuration commands (vector/IRQ setup). Suggests these were stripped from retail release for stability/security, retained only in developer/SpUtil builds.

## Data Flow Through This File

1. **Initialization sequence**: 
   - Caller issues `TSR_DRIVER_OPEN` (command 0x1 | TSR_NEED_PACKET) with `TsrDrvOpenPacket`
   - Driver populates response: copyright, version, device count
   - Game engine knows how many input devices exist

2. **Per-device activation**:
   - `TSR_DEVICE_OPEN` → get device copyright/capabilities
   - `TSR_DEVICE_ENABLE` → activate polling for that device

3. **Polling loop** (game tick):
   - `TSR_DEVICE_GETFORCE` → populate `TsrForcePacket` with 6-DOF forces + timestamp
   - `TSR_DEVICE_GETBUTTONS` → populate `TsrButtonPacket` with button state
   - Game engine reads packet fields and applies input

4. **Magic validation**: Caller *must* set `packet.command.data = 0xFF0000FF` before each call. Driver rejects packets with invalid magic. This prevents accidental/corrupt packet handling.

## Learning Notes

**Why this design matters**: This header captures how pre-Windows games interfaced with optional hardware. No abstraction layer could hide the x86 register protocol — TSR drivers were a quasi-standard, but each device had its own interrupt vector. The magic validation and device enumeration pattern persisted in modern USB and HID stacks.

**Era-specific idioms**:
- No memory allocation (fixed packet sizes, stack-based communication)
- No callbacks or async I/O (synchronous polling via interrupt returns)
- Error codes split across register halves (AH=major, AL=minor) to fit 16-bit status
- Conditional compilation guarding private APIs suggests this was a real shipping product with internal/debug variants

**Connection to modern engine concepts**: This is an early, minimalist **input abstraction layer** — a precursor to modern input systems (Xinput, Raw Input). The packet union pattern is similar to how game engines handle polymorphic input events today, just via interrupts instead of callbacks.

## Potential Issues

- **Weak authentication**: Magic value `0xFF0000FF` is a constant, not cryptographic. A corrupt packet or software bug could trick the driver into misinterpreting data.
- **No version negotiation**: Driver and client assume protocol compatibility; future TSR versions with extended fields would break silently.
- **Dead code**: Commented-out error codes (`#if 0` block, lines ~151–165) and `PRIVATE_STRUCTS` suggest incomplete cleanup or incomplete removal. The `SPWTSRERR_*` codes below that are active, creating redundancy.
- **Non-portable**: Hardcoded x86 real-mode semantics; cannot run on modern 64-bit systems or non-x86 architectures without emulation.
