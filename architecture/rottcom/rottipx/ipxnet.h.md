# rottcom/rottipx/ipxnet.h

## File Purpose
IPX (Internetwork Packet Exchange) network protocol header defining packet structures, node addressing, and core networking functions for multiplayer communication in the ROTT game engine. Bridges game data with low-level IPX driver primitives.

## Core Responsibilities
- Define IPX packet header format and fields (destination/source network/node/socket)
- Define Event Control Block (ECB) for IPX driver communication
- Provide game-specific payload wrapper (`rottdata_t`) and setup data structure
- Declare network initialization and shutdown functions
- Declare packet send/receive interface functions
- Manage node address registry and time-stamping for packet sequencing

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `rottdata_t` | typedef struct | Game payload wrapper (private data within `MAXPACKETSIZE` bytes) |
| `setupdata_t` | struct | Multiplayer setup handshake data (client ID, player number, command, player count) |
| `IPXPacket` | struct | IPX network layer header (checksum, length, transport control, destination/source network/node/socket) |
| `localadr_t` | struct | Local network address (network ID + node ID) |
| `nodeadr_t` | struct | Remote node address (6-byte node ID) |
| `ECB` | struct | Event Control Block for IPX driver (in-use flag, completion code, socket, workspace, fragments) |
| `packet_t` | struct | Complete packet structure (ECB + IPX header + timestamp + game data) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `nodeadr` | `nodeadr_t[]` | extern (global) | Registry of remote node addresses indexed by node number (up to `MAXNETNODES+1`) |
| `localnodenum` | int | extern (global) | Local node identifier |
| `ipxlocaltime` | long | extern (global) | Local timestamp for outgoing packets (sequence ordering) |
| `remotetime` | long | extern (global) | Timestamp of last received packet |
| `remoteadr` | `nodeadr_t` | extern (global) | Current remote node address for active peer |

## Key Functions / Methods

### InitNetwork
- **Signature:** `void InitNetwork(void)`
- **Purpose:** Initialize the network subsystem and prepare for IPX communication.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes global state (`nodeadr`, `localnodenum`, socket registration with IPX driver).
- **Calls:** Not visible in this file.
- **Notes:** Must be called before any send/receive operations.

### ShutdownNetwork
- **Signature:** `void ShutdownNetwork(void)`
- **Purpose:** Gracefully close network subsystem and release IPX resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Closes sockets, deallocates ECBs, clears global state.
- **Calls:** Not visible in this file.
- **Notes:** Cleanup only; no packet operations allowed after this.

### SendPacket
- **Signature:** `void SendPacket(int destination)`
- **Purpose:** Transmit a game packet to a destination node.
- **Inputs:** `destination` – node index to receive packet
- **Outputs/Return:** None (packet queued to IPX driver)
- **Side effects:** Updates `ipxlocaltime`; enqueues ECB for transmission; modifies packet buffer.
- **Calls:** Not visible in this file.
- **Notes:** Up to `MAXPACKETSPERPLAYER` outstanding packets per player before loss; caller must populate `rottdata_t` before calling.

### GetPacket
- **Signature:** `int GetPacket(void)`
- **Purpose:** Retrieve a received packet from the network queue.
- **Inputs:** None
- **Outputs/Return:** Nonzero if packet available; zero if queue empty
- **Side effects:** Updates `remotetime`, `remoteadr`; dequeues ECB from driver; copies data into buffer.
- **Calls:** Not visible in this file.
- **Notes:** Returns multiple packets if more than one is waiting; sequenced by timestamp; up to `NUMPACKETS` outstanding before loss.

## Control Flow Notes
- **Initialization phase:** `InitNetwork()` called at game startup to register local node and open sockets.
- **Frame/network loop:** Per-frame calls to `GetPacket()` drain incoming queue; `SendPacket()` queues outgoing messages.
- **Shutdown phase:** `ShutdownNetwork()` called at game exit to release IPX resources.
- Packets are time-stamped to ensure ordering when multiple arrivals are pending.

## External Dependencies
- **`c:\merge\rottnet.h`** (absolute path; likely a merged shared header) — defines `MAXPACKETSIZE`, `MAXNETNODES`, and other constants
- **`global.h`** — provides base type definitions (`BYTE`, `WORD`, `boolean`)
- **Defined elsewhere:** `MAXPACKETSIZE`, `MAXNETNODES` constants; implementation of `InitNetwork`, `ShutdownNetwork`, `SendPacket`, `GetPacket`
