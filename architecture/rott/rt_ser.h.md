# rott/rt_ser.h

## File Purpose
Serial modem communication interface for multiplayer networking. Provides setup, teardown, and packet I/O functions for players connected via serial/modem links in networked gameplay.

## Core Responsibilities
- Initialize and shutdown modem-based multiplayer game sessions
- Read incoming serial packets into a global buffer
- Write outgoing packets to the serial port
- Manage global packet data state

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `serialpacket` | `char[MAXPACKET]` | global | Receive buffer for incoming serial packets |
| `serialpacketlength` | `int` | global | Current length of data in `serialpacket` |

## Key Functions / Methods

### SetupModemGame
- Signature: `void SetupModemGame(void)`
- Purpose: Initialize modem/serial link for multiplayer game
- Inputs: None
- Outputs/Return: None
- Side effects: Configures serial hardware, enables interrupt handlers, initializes packet buffers
- Calls: Not inferable from this file
- Notes: Must be called before any `ReadSerialPacket` or `WriteSerialPacket` operations

### ShutdownModemGame
- Signature: `void ShutdownModemGame(void)`
- Purpose: Clean up modem/serial resources and disconnect
- Inputs: None
- Outputs/Return: None
- Side effects: Closes serial connection, disables interrupts, frees allocated resources
- Calls: Not inferable from this file
- Notes: Called on game exit or when ending multiplayer session

### ReadSerialPacket
- Signature: `boolean ReadSerialPacket(void)`
- Purpose: Attempt to read incoming packet from serial port
- Inputs: None (reads from serial hardware)
- Outputs/Return: `boolean` — true if packet received, false otherwise
- Side effects: Populates global `serialpacket` buffer and updates `serialpacketlength`; reads from serial I/O
- Calls: Not inferable from this file
- Notes: Blocking/non-blocking behavior unknown from signature alone

### WriteSerialPacket
- Signature: `void WriteSerialPacket(char *buffer, int len)`
- Purpose: Transmit packet data via serial port
- Inputs: `buffer` — packet payload pointer; `len` — byte count
- Outputs/Return: None
- Side effects: Writes data to serial hardware
- Calls: Not inferable from this file
- Notes: No return value; error handling behavior unclear

## Control Flow Notes
Fits into multiplayer network layer:
- **Init:** `SetupModemGame()` called before entering multiplayer game loop
- **Frame/Update:** `ReadSerialPacket()` polls for opponent state; `WriteSerialPacket()` sends local player updates
- **Shutdown:** `ShutdownModemGame()` on session end

## External Dependencies
- `rt_def.h`: Supplies `boolean` typedef, standard types (`char`, `int`)
- `MAXPACKET`: Constant defined elsewhere (likely `rt_def.h`)
- Serial hardware/ISR: Modem interrupt handlers and driver layer (implementation not visible)
