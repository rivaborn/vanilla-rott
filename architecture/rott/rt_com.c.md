# rott/rt_com.c

## File Purpose
Network communication and time synchronization module for Rise of the Triad's multiplayer system. Handles packet I/O through DOS interrupt calls to a real-mode network driver, and implements a 5-phase clock synchronization protocol between master (server) and slave (client) players.

## Core Responsibilities
- Initialize network interface and locate shared real-mode COM structure
- Read and write game packets with CRC integrity checks
- Manage 5-phase time synchronization handshake between master and slave
- Calculate and store round-trip transit times for latency compensation
- Handle server/client role transitions in packet addressing
- Validate incoming sync packets and coordinate phase progression
- Broadcast start commands and wait for player readiness during game init

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `syncpackettype` | struct | Sync control packet (type, phase, clocktime, delta, payload) |
| `synctype` | struct | Wrapper for sync state (sendtime, deltatime, packet) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rottcom` | `rottcom_t*` | global | Pointer to shared real-mode COM structure for driver I/O |
| `badpacket` | int | global | Flag: 1 if last packet had CRC mismatch |
| `consoleplayer` | int | global | Local player index |
| `ROTTpacket` | byte[] | global | Receive buffer for incoming network packets |
| `controlsynctime` | int | global | Reference timestamp for game start sync |
| `comregs` | union REGS | static | CPU registers for `int386()` interrupt calls |
| `ComStarted` | int | static | Guard flag to prevent re-initialization |
| `transittimes` | int[] | static | Per-player round-trip latency in tics (MAXPLAYERS entries) |

## Key Functions / Methods

### InitROTTNET
- **Signature:** `void InitROTTNET(void)`
- **Purpose:** Initializes the network driver interface by parsing command-line arguments to locate the shared COM structure address.
- **Inputs:** None (reads from `_argv` and command-line parameters)
- **Outputs/Return:** None
- **Side effects:** Sets `ComStarted=true`, reads `rottcom` pointer, prints debug info, sets `remoteridicule` flag based on rottcom->ticstep
- **Calls:** `CheckParm()`, `atol()`, `printf()`
- **Notes:** Only executes once per session due to `ComStarted` guard. Command-line must include `-net <address>` with the real-mode shared structure address.

### ReadPacket
- **Signature:** `boolean ReadPacket(void)`
- **Purpose:** Polls the network driver for an incoming packet, validates CRC, and copies to `ROTTpacket` if valid.
- **Inputs:** None (reads from `rottcom` structure via driver)
- **Outputs/Return:** `true` if packet received and valid, `false` if no packet ready
- **Side effects:** Calls `int386()` with CMD_GET, sets `badpacket=1` on CRC mismatch, copies packet data to `ROTTpacket`, adjusts `remotenode` for server/client mode, prints soft errors
- **Calls:** `int386()`, `CalculateCRC()`, `SoftError()`, `memcpy()`
- **Notes:** Sets `badpacket` flag but still returns true if packet arrived; caller must check `badpacket`. Adjusts remote node index based on server/client mode.

### WritePacket
- **Signature:** `void WritePacket(void *buffer, int len, int destination)`
- **Purpose:** Sends a game packet to a destination player via the network driver, with CRC appended.
- **Inputs:** `buffer` (packet data), `len` (payload length), `destination` (player index or server flag)
- **Outputs/Return:** None
- **Side effects:** Calls `int386()` with CMD_SEND, appends CRC to packet, increments `remotenode` if server (for addressing), memcpy's payload into driver buffer
- **Calls:** `int386()`, `CalculateCRC()`, `memcpy()`, `Error()`
- **Notes:** Errors if packet exceeds `MAXCOMBUFFERSIZE`. CRC occupies last 2 bytes after payload. Server mode increments destination node for driver fix-up.

### ValidSyncPacket
- **Signature:** `boolean ValidSyncPacket(synctype *sync)`
- **Purpose:** Attempts to read and validate an incoming sync packet.
- **Inputs:** `sync` pointer (output destination for valid packet)
- **Outputs/Return:** `true` if packet received, had no CRC error, and type is COM_SYNC
- **Side effects:** Calls `ReadPacket()`, copies packet into `sync->pkt` if valid
- **Calls:** `ReadPacket()`, `memcpy()`
- **Notes:** Returns false if `ReadPacket()` returns false or `badpacket==1`.

### SendSyncPacket
- **Signature:** `void SendSyncPacket(synctype *sync, int dest)`
- **Purpose:** Sends a sync packet with current tic count as send time.
- **Inputs:** `sync` (contains packet to send), `dest` (destination player)
- **Outputs/Return:** None
- **Side effects:** Sets `sync->sendtime=ticcount`, sets packet type to COM_SYNC, calls `WritePacket()`
- **Calls:** `WritePacket()`
- **Notes:** Caller must set `sync->pkt.phase` before calling this function.

### SlavePhaseHandler
- **Signature:** `boolean SlavePhaseHandler(synctype *sync)`
- **Purpose:** Processes incoming sync packet phases on client side and adjusts local clock.
- **Inputs:** `sync` (received sync packet with phase field set)
- **Outputs/Return:** `true` if sync complete (phase 5), `false` otherwise
- **Side effects:** Calls `ISR_SetTime()` to adjust system clock based on phase (2, 4, 5); updates `sync->sendtime` on completion
- **Calls:** `ISR_SetTime()`
- **Notes:** Implements phases 1–5 of client-side sync. Phase 1 is no-op; phase 5 marks completion.

### MasterPhaseHandler
- **Signature:** `boolean MasterPhaseHandler(synctype *sync)`
- **Purpose:** Advances sync packet phases on server side based on client responses.
- **Inputs:** `sync` (contains current phase and timing deltas)
- **Outputs/Return:** `true` if sync complete (phase 5), `false` otherwise
- **Side effects:** Increments phase, computes `delta` values, updates packet fields, sets `sendtime`
- **Calls:** None
- **Notes:** Implements phases 1–5 of server-side sync. Calculates round-trip delta and adjusts clocks accordingly.

### SetTime
- **Signature:** `void SetTime(void)`
- **Purpose:** Perform initial game-start synchronization: master sends start command, clients acknowledge, all wait for common time.
- **Inputs:** None (reads global `networkgame`, `IsServer`, `consoleplayer`, `numplayers`)
- **Outputs/Return:** None
- **Side effects:** Calls `SyncTime()` for each remote player, broadcasts COM_START packets, prints debug messages, adds UI message ("All players synched")
- **Calls:** `SafeMalloc()`, `PlayerInGame()`, `SyncTime()`, `WritePacket()`, `ReadPacket()`, `AddMessage()`, `ThreeDRefresh()`, `SafeFree()`, `AbortCheck()`
- **Notes:** Server sends COM_START twice with delay; clients wait for it. Flushes extra packets after sync. Sets global `controlsynctime`.

### SyncTime
- **Signature:** `void SyncTime(int client)`
- **Purpose:** Main 5-phase time synchronization loop; master initiates and guides phases, slave responds.
- **Inputs:** `client` (player index to sync with)
- **Outputs/Return:** None
- **Side effects:** Allocates `synctype`, alternates between `InitialMasterSync`/`InitialSlaveSync`, runs phase loop, calls `SetTransitTime()` on master, logs debug messages
- **Calls:** `SafeMalloc()`, `InitialMasterSync()`, `InitialSlaveSync()`, `ValidSyncPacket()`, `CalculateCRC()`, `MasterPhaseHandler()`, `SlavePhaseHandler()`, `SendSyncPacket()`, `ReadPacket()`, `SetTransitTime()`, `Error()`, `AbortCheck()`, `SafeFree()`
- **Notes:** Master sends every SYNCTIME tics; slave waits for packet. Calculates average delta time across all phases. Role determined by `networkgame && IsServer` (master) vs. otherwise (slave).

### InitialMasterSync
- **Signature:** `void InitialMasterSync(synctype *sync, int client)`
- **Purpose:** Master's initial handshake with a client, announces other clients being synced (SYNC_MEMO phase).
- **Inputs:** `sync`, `client` (index of client to sync)
- **Outputs/Return:** None
- **Side effects:** Sends SYNC_MEMO packets to clients with index > current client, waits for client to respond with SYNC_PHASE0, busywaits for timing
- **Calls:** `SendSyncPacket()`, `ValidSyncPacket()`, `AbortCheck()`
- **Notes:** Informs already-connected clients about new player joining (for display purposes).

### InitialSlaveSync
- **Signature:** `void InitialSlaveSync(synctype *sync)`
- **Purpose:** Client's initial handshake response, displays server sync status message.
- **Inputs:** `sync`
- **Outputs/Return:** None
- **Side effects:** Waits for SYNC_MEMO or SYNC_PHASE0 packets, displays messages via `AddMessage()`, sends response packet on SYNC_PHASE0
- **Calls:** `ValidSyncPacket()`, `AddMessage()`, `ThreeDRefresh()`, `SendSyncPacket()`, `ReadPacket()`, `AbortCheck()`, `itoa()`, `strcat()`
- **Notes:** Builds and displays "Server is synchronizing player N" message from `sync->pkt.clocktime`.

### SetTransitTime
- **Signature:** `void SetTransitTime(int client, int time)`
- **Purpose:** Records the measured round-trip latency for a player.
- **Inputs:** `client` (player index), `time` (latency in tics)
- **Outputs/Return:** None
- **Side effects:** Updates `transittimes[client]`
- **Calls:** None
- **Notes:** Called by master after each `SyncTime()` with half the average delta.

### GetTransitTime
- **Signature:** `int GetTransitTime(int client)`
- **Purpose:** Retrieves the stored round-trip latency for a player.
- **Inputs:** `client` (player index)
- **Outputs/Return:** Latency in tics
- **Side effects:** None
- **Calls:** None
- **Notes:** Returns value previously set by `SetTransitTime()`.

## Control Flow Notes
- **Init phase:** `InitROTTNET()` called once at startup to establish driver connection.
- **Game start:** `SetTime()` invoked to synchronize all players' clocks before gameplay begins. Internally calls `SyncTime()` for each remote player.
- **Frame/update:** `ReadPacket()` called regularly by main game loop to receive incoming network messages.
- **Ongoing:** Individual message handlers (not in this file) process received packets; `SyncTime()` may be called periodically to re-sync if drift is detected.
- **Shutdown:** Implicit; `rottcom` pointer becomes invalid when network driver unloads.

## External Dependencies
- **DOS/System headers:** `<dos.h>`, `<conio.h>`, `<process.h>`, `<bios.h>` (real-mode interrupt interface)
- **Game headers:** 
  - `rt_def.h` — global constants and types
  - `rt_util.h` — utilities (malloc, error, command-line parsing)
  - `rt_crc.h` — `CalculateCRC()`
  - `isr.h` — `ISR_SetTime()`
  - `rt_main.h` — `ticcount` global
  - `rt_playr.h` — `PlayerInGame()`, `numplayers`
  - `rottnet.h` — `rottcom_t`, network constants
  - `rt_msg.h` — `AddMessage()`, `MSG_SYSTEM`
  - `rt_draw.h` — `ThreeDRefresh()`
- **External symbols (defined elsewhere):**
  - `int386()` — DOS real-mode interrupt dispatcher
  - `ticcount` — global game time counter (in tics)
  - `networkgame`, `IsServer`, `standalone` — global game mode flags
  - `quiet` — debug flag
  - `server` — server player index constant
  - `_argv[]`, `_argc` — command-line arguments
