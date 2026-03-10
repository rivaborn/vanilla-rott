# rottcom/rottipx/ipxnet.c — Enhanced Analysis

## Architectural Role

This file implements the low-level IPX transport layer that handles asynchronous packet send/receive for ROTT's multiplayer networking. It bridges the game engine's application-level network API (via the `rottcom` global structure) and the DOS-resident IPX driver, providing a polling-based, timestamp-ordered packet queue. The module is fundamentally a **hardware-abstraction driver** that isolates the rest of the engine from DOS/IPX-specific interrupt and register manipulation details.

Based on the presence of `ControlPlayerObj`, `AddClientPacket`, and other net functions in the cross-references, there is almost certainly a higher-level coordination layer (likely `rott/rt_net.c`) that calls `InitNetwork()`, `SendPacket()`, and repeatedly polls `GetPacket()` in the main game loop.

## Key Cross-References

### Incoming (who depends on this file)

- **Game network coordinator** (presumed `rott/rt_net.c` or similar): calls `InitNetwork()` at startup, `SendPacket(destination)` to unicast or broadcast packets, `GetPacket()` in the polling loop, `ShutdownNetwork()` at exit.
- **IPX setup module** (`ipxsetup.h` imports): provides `socketid`, `server`, `numnetnodes` globals that control packet allocation and mode (server vs client).
- **Global application state** (`rottcom` struct): acts as the data exchange buffer between game logic and transport layer (fields: `.data`, `.datalength`, `.remotenode`, `.numplayers`).

### Outgoing (what this file depends on)

- **DOS IPX driver** (interrupt 0x2F via `geninterrupt()`, function pointer `IPX()`): provides socket operations, packet listen/send, and address lookup.
- **Header definitions** (`rottnet.h`, `ipxnet.h`, `ipxsetup.h`): define `packet_t`, `ECB`, `IPXPacket`, `localadr_t`, `nodeadr_t` structures and constants.
- **Standard C runtime** (`malloc`, `memset`, `memcpy`, `memcmp`, `free`).
- **DOS register access** (`<dos.h>`, `_AX`, `_BX`, `_SI`, `_ES`, etc.): register convention for interrupt calls.

## Design Patterns & Rationale

1. **Interrupt-Driven Async I/O with Polling Completion**: The IPX driver receives packets asynchronously via ECBs (Event Control Blocks), but the application polls the `InUseFlag` field to detect completion. This is idiomatic for DOS-era networking: no callbacks, no threads, just polling state flags.

2. **Ring Buffer with Timestamp Ordering**: `GetPacket()` scans all registered buffers to find the packet with the **lowest timestamp**, ensuring packets are delivered in send order even if the IPX driver completes them out of sequence. This is essential for deterministic multiplayer synchronization.

3. **Separate Send and Receive Buffers**: `packets[0]` is reserved for send; `packets[1..numnetpackets-1]` are pre-registered with the driver for async receive. This avoids buffer reuse race conditions.

4. **Dynamic Allocation by Server Role**: Servers allocate `NUMPACKETSPERPLAYER * numnetnodes` buffers (higher traffic), clients allocate `NUMPACKETS` (lower). This reflects the client-server architecture where the server relays all traffic.

5. **Broadcast Node Special Case**: The array index `MAXNETNODES` (not a real player) holds the broadcast MAC address (all `0xff`), allowing `SendPacket(MAXNETNODES)` to implement single-call broadcast.

**Rationale for this design**: In 1994–95, DOS was the standard gaming platform. Direct hardware/DOS API access was unavoidable. Polling is simple and predictable; timestamp-ordered delivery avoids the complexity of in-order guarantees at the driver level. The design reflects constraints of the era: no RTOS, no virtual memory, single-threaded execution.

## Data Flow Through This File

**Outbound:**
```
Game logic populates: rottcom.data[...], rottcom.datalength
        ↓
SendPacket(nodeindex) called
        ↓
Timestamp packets[0]→time with ipxlocaltime
Copy destination node address from nodeadr[nodeindex]
Set IPX header (destination network/node/socket)
Call IPX interrupt (BX=3, send ECB)
        ↓
Poll packets[0]→ecb.InUseFlag until zero
(Interleaved IPX relinquish control (BX=10) to yield CPU)
        ↓
Packet sent to network
```

**Inbound:**
```
IPX driver delivers packet asynchronously into packets[i]→ipx, sets ecb.InUseFlag=0
        ↓
Game loop calls GetPacket()
        ↓
Scan packets[1..numnetpackets-1], find lowest timestamp (skip if InUseFlag still set)
Extract sender node from packets[i]→ipx.sNode
Look up sender in nodeadr[] array → populate rottcom.remotenode
Extract payload: ShortSwap packet length, memcpy into rottcom.data
        ↓
Re-register ECB via ListenForPacket() (async receive queue)
        ↓
Return 1; game logic processes rottcom.data[...] and rottcom.datalength
```

**Special case:** Broadcast packets with `time == -1` are skipped if local time is valid (assumed to be setup/sync broadcasts from other games on the network; not relayed to game logic).

## Learning Notes

1. **1990s DOS Networking Idioms:**
   - No sockets API; direct DOS interrupt (0x2F / IPX). Modern engines use socket abstractions.
   - Polling-based async I/O instead of callbacks or threads. Modern engines use event loops or thread pools.
   - Fixed-size packet arrays instead of dynamic queues. Reflects memory constraints and predictable allocation patterns.

2. **Deterministic Multiplayer Design:**
   - Timestamp-based ordering ensures deterministic packet delivery order even over unreliable transport (IPX datagram). This is crucial for replaying multiplayer games.
   - Server acts as a de facto message ordering authority by allocating more buffers and likely relaying packets.

3. **Hardware Abstraction:**
   - The module cleanly separates IPX protocol details (ECB setup, interrupt register manipulation, byte swapping) from game logic. Game code only sees `rottcom` struct and four public functions.

4. **Node Addressing:**
   - The module maintains a hardcoded lookup table `nodeadr[]` with indices 0=local, 1..n=remote players, MAXNETNODES=broadcast. This assumes a fixed player set, consistent with match-based gameplay.

5. **Resource Allocation Strategy:**
   - Servers allocate more buffers per player to handle fan-out (relaying). This is a scaling heuristic for the era.

6. **No Explicit Error Recovery:**
   - Packets that fail to send (IPX error) abort with `Error()`. Packets that fail to receive (bad completion code) are similarly aborted. No retry or timeout logic; designed for LAN reliability.

## Potential Issues

1. **Busy-Wait in SendPacket:** The spin loop `while(packets[0]->ecb.InUseFlag != 0)` blocks the entire game until the packet is sent. On slow networks or if the IPX driver stalls, this could cause frame rate hiccups. The "relinquish control" call (BX=10) helps but doesn't eliminate the stall.

2. **Hard-Coded Buffer Limits:** `MAXPACKETS` and `MAXNETNODES` are compile-time constants. If exceeded, the module silently caps allocation, potentially dropping packets.

3. **No Timeout/Retry for Hung Packets:** If the IPX driver never sets `InUseFlag=0`, or if a received packet's `CompletionCode` is non-zero, the code aborts immediately. No graceful degradation or fallback.

4. **Implicit Single-Threaded Assumption:** The polling loop assumes no concurrent access. If the game engine ever spawns threads or uses interrupts for other purposes, race conditions are possible (e.g., IPX driver modifying `InUseFlag` while game thread reads it).

5. **Broadcast Sync Heuristic is Fragile:** Packets with `time == -1` are assumed to be out-of-band setup broadcasts. If legitimate data packets happen to have this timestamp, they are silently discarded. No explicit protocol marker to distinguish broadcast sync from data.

---

**Note:** The ARCHITECTURE CONTEXT was incomplete (max turns reached), so cross-references to higher-level callers are inferred from function signatures and the presence of `rottcom`/`rt_net.c` in the project. A complete subsystem map would reveal the exact call sites and usage patterns.
