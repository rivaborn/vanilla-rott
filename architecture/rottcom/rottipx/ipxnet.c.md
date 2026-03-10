# rottcom/rottipx/ipxnet.c

## File Purpose
Implements the IPX (Internetwork Packet Exchange) network driver for the ROTT multiplayer engine. Provides packet allocation, socket management, and send/receive operations via interrupt-driven IPX protocol calls. Handles node discovery and packet sequencing by timestamp.

## Core Responsibilities
- Allocate and deallocate packet buffers dynamically based on server/client role
- Initialize IPX driver detection and socket creation
- Register packet receive buffers with the IPX driver via ECBs (Event Control Blocks)
- Send data packets to remote nodes (unicast or broadcast)
- Receive incoming packets, identify sender, and extract application data
- Maintain a node address lookup table and route packets by node index
- Perform byte-order conversions for network fields

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `packet_t` | struct | Complete packet unit: ECB + IPX header + timestamp + payload |
| `IPXPacket` | struct | IPX protocol header with network/node/socket addressing |
| `ECB` | struct | Event Control Block; driver interface for send/recv operations |
| `localadr_t` | struct | Local network address (4-byte network + 6-byte node ID) |
| `nodeadr_t` | struct | Remote node address (6-byte node ID only) |
| `rottdata_t` | struct | Opaque application data payload buffer |
| `setupdata_t` | struct | Setup-phase command structure (client/player/command fields) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `packets[MAXPACKETS]` | `packet_t*[]` | global | Pointers to allocated send/recv packet structures |
| `nodeadr[MAXNETNODES+1]` | `nodeadr_t[]` | global | LUT: index 0=local, 1..n=remote nodes, MAXNETNODES=broadcast |
| `remoteadr` | `nodeadr_t` | global | Sender address of last received packet |
| `localadr` | `localadr_t` | global | Local network/node address (set at init) |
| `IPX` | `void far (*)()` | global | Function pointer to IPX interrupt handler |
| `ipxlocaltime` | `long` | global | Timestamp stamped into outgoing packets |
| `remotetime` | `long` | global | Timestamp of most recently received packet |
| `numnetpackets` | `int` | global | Runtime count of allocated packets |

## Key Functions / Methods

### InitNetwork
- **Signature:** `void InitNetwork(void)`
- **Purpose:** Detect IPX driver, obtain function pointer, allocate packet buffers, open a socket, retrieve local address, and register all recv buffers with the driver.
- **Inputs:** None (reads `socketid`, `server`, `numnetnodes` from globals/externs).
- **Outputs/Return:** None; side effects populate `IPX`, `socketid`, `localadr`, `nodeadr[0]`, `nodeadr[MAXNETNODES]`.
- **Side effects:** Calls `geninterrupt(0x2f)` to detect IPX; allocates `numnetpackets` packet structures; modifies global socket and address tables; prints to stdout.
- **Calls:** `AllocatePackets()`, `OpenSocket()`, `GetLocalAddress()`, `ListenForPacket()` (×numnetpackets−1).
- **Notes:** Aborts with `Error()` if IPX not detected or socket open fails. Calls `ListenForPacket()` to pre-register recv buffers; packet[0] is reserved for send. Byte-swaps socket ID (big-endian to little-endian).

### SendPacket
- **Signature:** `void SendPacket(int destination)`
- **Purpose:** Queue an outgoing packet to a specific node (or broadcast if destination == MAXNETNODES).
- **Inputs:** `destination` = node index or MAXNETNODES for broadcast.
- **Outputs/Return:** None; packet transmitted when `SendPacket()` returns.
- **Side effects:** Stamps `packets[0]->time` with `ipxlocaltime`; sets destination node address in `packets[0]->ipx.dNode` and `packets[0]->ecb.ImmediateAddress`; polls until `InUseFlag` clears (busy-wait). Calls IPX function (BX=3).
- **Calls:** Indirect IPX call via function pointer.
- **Notes:** Caller must populate `rottcom.data` and `rottcom.datalength` before calling. Polls driver with IPX relinquish control (BX=10) to yield CPU on polled drivers. Aborts if IPX returns error.

### GetPacket
- **Signature:** `int GetPacket(void)`
- **Purpose:** Poll for completed recv packets, return oldest by timestamp, and identify sender.
- **Inputs:** None (reads packet ECBs and timestamps).
- **Outputs/Return:** Returns 1 if packet received and copied to `rottcom.data`/`rottcom.datalength`, 0 otherwise.
- **Side effects:** Sets `rottcom.remotenode` (node index of sender or −1), `remotetime`, and `remoteadr`. Repostings ECB via `ListenForPacket()` after extraction.
- **Calls:** `ListenForPacket()` (re-registration), `ShortSwap()`, `memcpy()`, `memcmp()`.
- **Notes:** Scans packets[1..numnetpackets−1] for lowest timestamp; ignores in-use packets. Skips broadcast sync packets (time == −1 and local time != −1). Checks `CompletionCode` for errors. Extracts sender from `packet->ipx.sNode` and matches against `nodeadr[]` LUT to populate `rottcom.remotenode`. Returns 0 without copying if packet is out-of-order setup broadcast.

### AllocatePackets
- **Signature:** `void AllocatePackets(void)`
- **Purpose:** Dynamically allocate packet buffer pool based on server role.
- **Inputs:** Reads `server`, `numnetnodes` from globals.
- **Outputs/Return:** None; populates `packets[]` array.
- **Side effects:** Allocates `numnetpackets = server ? NUMPACKETSPERPLAYER * numnetnodes : NUMPACKETS`; caps at MAXPACKETS; zeros all structures.
- **Calls:** `malloc()`, `memset()`, `Error()`.
- **Notes:** Aborts if allocation fails.

### DeAllocatePackets
- **Signature:** `void DeAllocatePackets(void)`
- **Purpose:** Free all allocated packet structures.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `free()` on all non-NULL `packets[]` pointers.
- **Calls:** `free()`.

### OpenSocket, CloseSocket, ListenForPacket, GetLocalAddress
- **Purpose:** Thin wrappers around IPX interrupt calls (BX=0/1/4/9 respectively).
- **Notes:** Load CPU registers and invoke `IPX()`. These use DOS register calling convention (`_AX`, `_BX`, `_SI`, `_ES`, etc.). `OpenSocket`/`ListenForPacket` abort on error.

### ShortSwap
- **Signature:** `unsigned short ShortSwap(unsigned short i)`
- **Purpose:** Swap bytes for network byte-order conversion (big-endian ↔ little-endian).
- **Inputs:** 16-bit value.
- **Outputs/Return:** Swapped value.

## Control Flow Notes
- **Init phase:** `InitNetwork()` runs once, sets up driver and recv buffers.
- **Frame/polling phase:** Game loop calls `GetPacket()` repeatedly to drain recv queue, and `SendPacket()` to queue outgoing packets.
- **Shutdown phase:** `ShutdownNetwork()` closes socket and deallocates buffers.

Packet flow is event-driven: the IPX driver completes recv ECBs asynchronously; `GetPacket()` polls the `InUseFlag` to detect completion.

## External Dependencies
- **Includes/imports:** `<dos.h>`, `<process.h>`, `<values.h>` (DOS-specific headers for register access, interrupts); `rottnet.h`, `ipxnet.h`, `ipxsetup.h` (local headers).
- **Extern symbols:** `socketid`, `server`, `numnetnodes` (from `ipxsetup.h`); `rottcom` (global game state struct, used for `.data`, `.datalength`, `.numplayers`, `.remotenode`).
- **System calls:** `geninterrupt(0x2f)` (IPX driver detection via DOS interrupt).
- **Implicit dependency:** IPX driver installed and resident in DOS memory.
