# rott/rottser.h

## File Purpose
Header file defining serial port configuration data structure. Used to store and pass serial communication parameters (IRQ, UART base address, and baud rate) throughout the game engine.

## Core Responsibilities
- Define the serial port configuration structure
- Provide a standard type for serial device setup
- Encapsulate hardware-level serial communication parameters

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `serialdata_t` | struct | Holds serial port hardware configuration (IRQ number, UART port, baud rate) |

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This is a data structure header with no executable code. Likely used during initialization to configure serial devices (e.g., for modem support, network play, or debug output). The structure is instantiated elsewhere and passed to serial communication setup functions.

## External Dependencies
- Standard C library headers (implicitly included by files that use this header)
- No engine-specific dependencies visible

---

**Notes:**
- The three fields (`irq`, `uart`, `baud`) suggest DOS-era serial port configuration where IRQ levels and UART I/O addresses were manually specified
- `long` type for all fields suggests 32-bit values on the target platform
- This is part of a serial communication subsystem, likely for network multiplayer or modem connectivity
