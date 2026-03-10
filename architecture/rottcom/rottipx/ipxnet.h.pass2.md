# rottcom/rottipx/ipxnet.h — Enhanced Analysis

## Architectural Role
This header provides a thin **driver abstraction layer** between ROTT's game networking subsystem (in `rott/rt_net.c/h`) and the low-level IPX protocol driver. It encapsulates DOS-era Novell NetWare IPX primitives into game-friendly packet structures, enabling multiplayer communication via direct driver access. The design assumes a single IPX socket and time-stamped packet sequencing for reliable per-node delivery under lossy conditions (max 8 packets queued globally, 3 per player).

## Key Cross-References

### Incoming (who depends on this file)
- **`rott/rt_net.c/h`**: High-level multiplayer session management calls `SendPacket()` and `GetPacket()` in its frame loop; reads/writes `remoteadr` and global node registry to track active peers
- **`rottcom/rottipx/ipxnet.c`**: Implementation file; defines `InitNetwork()`, `ShutdownNetwork()`, packet pool via `AllocatePackets()`
- **`setupdata_t` consumers**: Game setup/lobby code populates setup handshake data during multiplayer init

### Outgoing (what this file depends on)
- **IPX driver (implicit)**: `SendPacket()` and `GetPacket()` delegate to driver-level ECB processing; driver is not exposed in source
- **`c:\merge\rottnet.h`** (absolute DOS path): Defines `MAXPACKETSIZE`, `MAXNETNODES` constants; this hardcoded path is a build portability issue
- **`global.h`**: Base integer/byte types (`WORD`, `BYTE`)

## Design Patterns & Rationale

**Wrapper Pattern**: `rottdata_t` wraps variable game payload in a fixed-size `MAXPACKETSIZE` buffer—needed because IPX MTU is fixed and DOS drivers require static buffer layouts.

**Struct Composition** (idiomatic 1990s driver API): `packet_t` combines ECB (driver control), IPXPacket (protocol header), timestamp, and payload in one contiguous struct. This mirrors DOS Novell API expectations where the driver directly fills the ECB and adjacent buffers.

**Time-Sequencing for Out-of-Order Handling**: Unlike modern sequence numbers in the protocol header, ROTT uses local system time (`ipxlocaltime`, `remotetime`) to order packets when multiple arrive simultaneously. This assumes:
- Network latency is low (LAN-only)
- System clock is monotonic within a game session
- No clock wraparound during typical game duration

**Registry-Based Node Tracking**: `nodeadr[MAXNETNODES+1]` is a static lookup table; callers pass node indices to `SendPacket()`, not full addresses. This implies a join/discovery phase populates the registry (likely via `setupdata_t` handshake).

## Data Flow Through This File

1. **Initialization**: `InitNetwork()` allocates packet pool, registers local IPX socket with driver, initializes node registry
2. **Send Path**: 
   - Game writes game state to `packet_t.data.private[0..MAXPACKETSIZE-1]`
   - Calls `SendPacket(node_index)` → driver enqueues ECB with destination from `nodeadr[node_index]`
   - `ipxlocaltime` stamped for sequencing
3. **Receive Path**: 
   - Driver delivers packets; ECB completion triggers callback (implicit, not in header)
   - `GetPacket()` polls queue, updates `remotetime` from packet, copies `remoteadr` (sender identity)
   - Game reads from `packet_t.data.private[]` and processes
4. **Shutdown**: `ShutdownNetwork()` flushes pending ECBs, deallocates pool

## Learning Notes

**Novell IPX Era**: This code is authentic DOS IPX—direct driver API with ECBs, explicit socket management, and no TCP/IP stack. Modern network code would use sockets API (winsock/BSD sockets) over TCP/IP with serialization libraries (protobuf, msgpack).

**Time-Stamping vs. Sequence Numbers**: ROTT uses wall-clock timestamps for packet ordering (elegant for LAN, fragile for long sessions or clock adjustments). Modern engines embed sequence numbers in the protocol header, independent of system time.

**No Acknowledgment/Retransmit**: The "8 packets before loss" limits suggest fire-and-forget UDP-like semantics; the driver likely drops old packets if the queue overflows. This is acceptable for frequent state updates (position, health) but risky for critical commands—the upper layer (`rt_net.c`) must handle resends.

**Single Socket Design**: All communication flows through one ECB/socket pair; no explicit per-peer connections. Multicast or broadcast may be possible via the IPX protocol but not exposed here.

## Potential Issues

- **Hardcoded DOS Path**: `#include "c:\merge\rottnet.h"` will fail on modern systems (non-existent drive/path). This blocks cross-platform compilation.
- **Timestamp Wraparound**: `long` timestamps for `ipxlocaltime` and `remotetime` will overflow after ~24 days on 32-bit systems; old packets could be misinterpreted as new.
- **No Bounds Checking**: `nodeadr[index]` access in `SendPacket()` is not guarded; caller must ensure `index ≤ MAXNETNODES`.
- **Missing Setup Semantics**: How `setupdata_t` populates `nodeadr[]` is opaque (defined in `ipxnet.c`); higher-level code must coordinate setup sequencing to avoid sending to uninitialized nodes.
