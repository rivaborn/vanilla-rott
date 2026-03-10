# rottcom/rottser/sercom.c

## File Purpose
Serial packet-based communication layer for multiplayer ROTT games. Implements frame-delimited packet I/O, connection negotiation between two players, and comprehensive statistics collection on serial link performance.

## Core Responsibilities
- **Packet I/O**: Read and write variable-length packets with frame character escaping over serial queue
- **Connection negotiation**: Synchronize two game clients via handshake protocol ("ROTT" packets with stage tracking)
- **Network ISR dispatch**: Route CMD_SEND and CMD_GET commands to packet handlers
- **Statistics tracking**: Count bytes, packets, buffer overruns, and UART errors; compute aggregate metrics
- **Timing**: Track game session start/end and elapsed playtime
- **State management**: Maintain escape state machine for packet framing and oversize packet detection

## Key Types / Data Structures
None defined in this file (uses external `rottcom` structure and `que_t` queues from port.h).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `starttime`, `endtime`, `playtime` | `time_t` | static | Session timing |
| `writeBufferOverruns`, `bytesRead`, `packetsRead`, `largestReadPacket`, `smallestReadPacket`, `readBufferOverruns`, `totalReadPacketBytes`, `oversizeReadPackets`, `largestOversizeReadPacket`, `overReadPacketLen` | `unsigned long` | static | Read statistics counters |
| `bytesWritten`, `packetsWrite`, `largestWritePacket`, `smallestWritePacket`, `totalWritePacketBytes`, `oversizeWritePackets`, `largestOversizeWritePacket` | `unsigned long` | static | Write statistics counters |
| `numBreak`, `numFramingError`, `numParityError`, `numOverrunError`, `numTxInterrupts`, `numRxInterrupts` | `unsigned long` | static | UART error counters |
| `packet[MAXPACKET]` | `char[]` | static | Incoming packet buffer |
| `localbuffer[MAXPACKET*2+2]` | `char[]` | static | Outgoing packet buffer (with escaping) |
| `packetlen`, `inescape`, `newpacket` | `int` | static | Packet parsing state machine |

## Key Functions / Methods

### ReadPacket
- **Signature:** `boolean ReadPacket(void)`
- **Purpose:** Read one complete frame-delimited packet from input queue. Implements escape-sequence state machine (0x70 as frame/escape marker).
- **Inputs:** Reads from global `inque` via `read_byte()` calls
- **Outputs/Return:** `true` if complete packet assembled in `packet[]` and `packetlen` set; `false` if incomplete
- **Side effects:** Updates `packetlen`, `inescape`, `newpacket` state; increments statistics counters (packetsRead, largestReadPacket, smallestReadPacket, totalReadPacketBytes, readBufferOverruns, oversizeReadPackets); may reset input queue on overflow
- **Calls:** `read_byte()` (defined elsewhere in port.c)
- **Notes:** Handles packet overruns silently (discards data > MAXPACKET); treats double FRAMECHAR (0x70 0x70) as literal byte; single FRAMECHAR followed by 0x00 terminates packet

### WritePacket
- **Signature:** `void WritePacket(char *buffer, int len)`
- **Purpose:** Escape and transmit a packet over serial link. Doubles any FRAMECHAR bytes (0x70 → 0x70 0x70) and appends frame terminator.
- **Inputs:** `buffer` (data to send), `len` (byte count)
- **Outputs/Return:** None (queues to output via `write_buffer()`)
- **Side effects:** Updates statistics (packetsWrite, largestWritePacket, smallestWritePacket, totalWritePacketBytes, oversizeWritePackets); silently discards packets > MAXPACKET
- **Calls:** `write_buffer()` (defined elsewhere in port.c)
- **Notes:** Oversize packets are counted but not transmitted; output format is escaped_data + FRAMECHAR(0x70) + NUL

### NetISR
- **Signature:** `void NetISR(void)`
- **Purpose:** Interrupt service routine dispatcher for network commands from global `rottcom` structure.
- **Inputs:** Reads `rottcom.command`, `rottcom.data`, `rottcom.datalength`
- **Outputs/Return:** None (updates `rottcom` fields: `remotenode`, `datalength`)
- **Side effects:** Modifies `rottcom.remotenode` and `rottcom.datalength`; calls `ReadPacket()` or `WritePacket()` as directed
- **Calls:** `ReadPacket()`, `WritePacket()`
- **Notes:** CMD_SEND transmits game data; CMD_GET waits for incoming packet (sets remotenode=1 on success, -1 on failure or oversized packet)

### Connect
- **Signature:** `int Connect(void)`
- **Purpose:** Negotiate connection between two game instances via synchronization handshake. Determines which player is player 0 or 1.
- **Inputs:** Command-line parameters (CheckParm for "-player" or "-answer"); polls keyboard for ESC abort
- **Outputs/Return:** `TRUE` on successful sync, `FALSE` on abort; sets global `rottcom.consoleplayer`
- **Side effects:** Sets `rottcom.consoleplayer` to 0 or 1; loops reading and writing "ROTT" packets until both sides reach stage 1
- **Calls:** `CheckParm()`, `bioskey()`, `ReadPacket()`, `WritePacket()`, `gettime()`, `delay()`, `printf()`
- **Notes:** Exchange format is "ROTT<player><stage>" (7 chars); if packet player matches local player, XOR consoleplayer and reset stages; loop exits when `remotestage >= 1`

### stats
- **Signature:** `void stats(void)`
- **Purpose:** Display comprehensive session and communication statistics on console.
- **Inputs:** Global timing and counter variables
- **Outputs/Return:** None (prints to stdout via printf, clears screen)
- **Side effects:** Calls `clrscr()`, `ctime()`, `showReadStats()`, `showWriteStats()`, `showUartErrors()`
- **Calls:** Timing functions, display functions listed below
- **Notes:** Skips time display if starttime == 0; computes playtime as endtime − starttime

### Helper functions (trivial, summary only):
- **StartTime / EndTime:** Call `time()` to record session boundaries
- **reset_counters:** Zero all 27 statistics counters
- **showReadStats / showWriteStats / showUartErrors:** Format and print statistics tables; handle zero denominators in average calculations

## Control Flow Notes
- **Initialization:** `Connect()` negotiates the link (likely called once at startup)
- **Frame loop:** `NetISR()` is invoked by serial interrupt handler; dispatches CMD_SEND/CMD_GET to read/write functions
- **Statistics:** `stats()` and `reset_counters()` are on-demand diagnostic functions (not in main game loop)
- **Shutdown:** No explicit cleanup in this file; relies on port.c `ShutdownPort()` for hardware teardown

## External Dependencies
- **port.h**: `read_byte()`, `write_buffer()`, `inque` (input queue), `outque` (output queue), `QUESIZE`, `MAXPACKETSIZE`
- **rottnet.h**: `rottcom` (global network command/data structure), `CMD_SEND`, `CMD_GET`
- **sersetup.h**: `showstats`, `usemodem` (globals referenced elsewhere, not used in sercom.c)
- **Standard C (DOS)**: `<time.h>` (time_t, time(), ctime()), `<conio.h>` (bioskey, clrscr), `<stdio.h>` (printf), `<string.h>` (memcpy, strncmp, sprintf, strlen), `<dos.h>` via port.h
- **Defined elsewhere (global.h):** `CheckParm()`, `Error()`, `delay()`, `gettime()`
