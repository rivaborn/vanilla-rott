# rott/tsr.h

## File Purpose
Header file defining the interface for communicating with TSR (Terminate and Stay Resident) device drivers via DOS interrupts. Defines data structures, command codes, and error codes for a hardware device driver protocol supporting force feedback and button input devices.

## Core Responsibilities
- Define packet structures for TSR driver/device initialization and data exchange
- Define interrupt command codes and flags for driver control operations
- Define error codes for driver operation failures
- Establish protocol constants for packet validation (magic data, timeouts)
- Support private driver configuration (conditionally compiled)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TsrDrvOpenPacket` | struct | Driver initialization response: copyright string, version (major/minor), device count |
| `TsrDevOpenPacket` | struct | Device initialization response: copyright string |
| `TsrForcePacket` | struct | Force feedback data: timestamp, button state, 6-DOF force/torque values (tx/ty/tz, rx/ry/rz) |
| `TsrButtonPacket` | struct | Button input data: timestamp, period, button state |
| `TsrCommandPacket` | struct | Command wrapper with magic validation data (0xFF0000FF) |
| `TsrPrivatePacket` | struct | Private driver control packet (conditional); wraps command + union of vector/hardware info |
| `TsrPacket` | union | Top-level packet union for all communication types (command, driver/device open, force, button, private) |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a header-only interface definition file.

## Control Flow Notes
Defines an interrupt-driven protocol for DOS TSR drivers. Callers:
1. Populate register AX with command code (e.g., `TSR_DRIVER_OPEN`)
2. Load DX with device number
3. If `TSR_NEED_PACKET` flag is set in command, load ES:BX with pointer to `TsrPacket` and ensure `packet.command.data == TSRCMD_DATA`
4. Issue interrupt; driver returns error code in AX (0 = success)

Public operations include driver/device open/close, device enable/disable, and force/button data retrieval. Private operations (guarded by `#ifdef PRIVATE_STRUCTS`) allow vector reconfiguration and physical hardware info queries.

## External Dependencies
- Conditional reference to `SPWERR_TSR` macro (defined elsewhere) used in error code construction
- No standard library includes
- Designed for 16-bit x86 DOS environment (register-level protocol using AX, DX, BX, ES, etc.)
