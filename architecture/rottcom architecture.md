# Subsystem Overview

## Purpose
The rottcom subsystem provides multiplayer networking infrastructure for Rise of the Triad, supporting both IPX (Internetwork Packet Exchange) network gaming and serial/modem point-to-point connections. It orchestrates node discovery, player synchronization, interrupt-driven packet I/O, and inter-process communication between the game engine and network hardware drivers.

## Key Files
| File | Role |
|------|------|
| **rottnet.c/h** | IPC bridge: manages shared `rottcom_t` struct, hooks interrupt vectors for async communication, launches main ROTT game executable with network parameters |
| **rottipx/ipxnet.c/h** | IPX driver interface: allocates ECBs, sends/receives packets via interrupt 0x2f, maintains node address tables |
| **rottipx/ipxsetup.c/h** | IPX setup: discovers nodes via broadcast, assigns player numbers, synchronizes multi-player initialization |
| **rottipx/global.c/h** | IPX utilities: error reporting, command-line parsing, shutdown coordination |
| **rottser/sercom.c/h** | Serial packet layer: frame-delimited packet I/O with escape handling, connection handshake, statistics tracking |
| **rottser/port.c/h** | UART driver: detects 8250/16550 chips, manages circular I/O queues, hooks IRQs, handles serial interrupts |
| **rottser/sermodem.c/h** | Modem control: AT command transmission, Hayes-compatible dial/answer/hangup sequences |
| **rottser/sersetup.c/h** | Serial setup: entry point for modem/serial mode, facilitates pre-game "talk mode", coordinates game launch |
| **rottser/st_cfg.c/h** | Configuration loader: parses SETUP.ROT/ROTT.ROT script files, extracts modem/serial parameters |
| **rottser/scriplib.c/h** | Script tokenizer: loads and parses configuration files, tracks line numbers and parse state |
| **rottser/serial.h** | UART constants: register offsets, bit masks for 16550-compatible hardware control |
| **rt_net.h** | Game protocol: defines packet types (COM_DELTA, COM_TEXT, COM_SYNC), game state structures, packet size utilities |

## Core Responsibilities
- Detect and initialize IPX or serial/modem hardware based on command-line arguments (server/client/modem dial/modem answer/serial modes)
- Discover available network nodes via IPX broadcast and assign player numbers (1–11 players, up to 14 nodes)
- Allocate, send, and receive variable-length packets with protocol-appropriate framing (IPX ECBs vs. serial escape sequences)
- Hook interrupt vectors and implement interrupt service routines for async communication (IPX via int 0x2f, UART via IRQ 3/4, network command vector 0x60–0x66)
- Parse command-line parameters and configuration files (SETUP.ROT, ROTT.ROT) for modem/serial settings
- Establish modem connections via AT commands or direct serial links and perform handshake synchronization
- Manage shared inter-process communication via `rottcom_t` struct for game engine ↔ network driver coordination
- Provide connection statistics and diagnostics (bytes sent/received, packet counts, UART errors, elapsed time)

## Key Interfaces & Data Flow
**Exposes to game engine:**
- `InitNetwork()`, `ShutdownNetwork()` — IPX layer init/cleanup
- `SendPacket(int dest)`, `GetPacket()` — packet send/receive (ipxnet module)
- `LaunchROTT()` — spawn main game executable with network parameters (rottnet module)
- `rottcom_t` struct — shared IPC block for game state (`data`, `datalength`, `numplayers`, `remotenode`, command codes `CMD_SEND`, `CMD_GET`)
- `rt_net.h` structures — game packet types for multiplayer state exchange

**Consumes from game engine:**
- Network commands (`CMD_SEND`, `CMD_GET`) via `rottcom.cmd`
- Game state payload in `rottcom.data` buffer (max `MAXPACKETSIZE`)

**Interacts with hardware and OS:**
- IPX driver (installed DOS TSR): sends/receives packets via ECB structures, interrupt 0x2f
- UART hardware (ports 0x3f8–0x2f8, IRQ 3–4): 8250/16550 serial ports for modem/direct serial communication
- Modem (Hayes-compatible): AT command interface for dialing, answering, disconnecting
- DOS: interrupt vectors, `spawnv()` to launch game executable, BIOS keyboard (`bioskey()`), clock (`time()`)
- Configuration files: SETUP.ROT, ROTT.ROT (script format, parsed by scriplib)

## Runtime Role
**Initialization phase:**
- Parse `rottcom` command-line to determine game mode (IPX vs. serial/modem, server vs. client, node count, phone number)
- `rottipx/ipxsetup.c` (IPX mode): Initialize IPX driver, broadcast node discovery packets, collect responses, assign player numbers
- `rottser/sersetup.c` (serial mode): Load configuration from scripts, initialize modem or serial port, dial/answer connection, perform handshake
- Display status messages and wait for player synchronization
- Populate node address tables and player-to-node mappings
- Launch main ROTT game executable via `spawnv()` with shared `rottcom` struct in place

**Frame phase (during gameplay):**
- ISR: receive incoming packets (via IPX ECB callback or UART IRQ), deposit in input queue or `rottcom.data`
- Game engine: poll `rottcom.cmd` for new packets, process, respond with outgoing data
- ISR: send outbound packets on demand (`CMD_SEND` command)
- Update connection statistics (byte counts, packet counts, error counters)

**Shutdown phase:**
- Game engine signals shutdown via command code
- `rottcom` cleanup: close sockets, restore interrupt vectors, flush remaining packets
- `rottser/sersetup.c`: optionally display connection statistics; hang up modem or close serial port
- Restore original interrupt handlers and DOS state

## Notable Implementation Details
- **IPX packet allocation:** Variable-sized ECB structures allocated at startup based on server/client role; reused throughout session to avoid heap fragmentation
- **Serial packet framing:** Frame delimiter (0x10) and escape character (0x11) with byte-stuffing to preserve transparency; detects oversize packets and resets on desync
- **UART detection:** Probes BIOS system data (0x0040:0x0000 – 0x0007) to identify COM port, IRQ, and UART type (8250 vs. 16550)
- **Circular queues:** Power-of-2 sizing with bitwise AND modulo for efficient wraparound (no division in ISR)
- **Two-stage handshake:** Serial modem: (1) modem connection (CONNECT response), (2) game handshake ("ROTT" packets with stage flags)
- **Byte-order conversion:** IPX network fields stored in big-endian; local operations convert to/from host byte order
- **Modem AT command pacing:** Character-level delays (50 ms) between command bytes and response polling loops to accommodate slow modems
- **Stack isolation for ISR:** rottnet.c allocates dedicated ISR stack and swaps during interrupt to prevent stack conflicts with game code
