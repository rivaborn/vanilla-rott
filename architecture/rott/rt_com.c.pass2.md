# rott/rt_com.c — Enhanced Analysis

## Architectural Role
This file is the **protected-mode/real-mode bridge** for ROTT's network subsystem. It translates high-level game packet I/O and time synchronization requests into DOS `int386()` calls to a real-mode network driver, managing the complex state machine for master-slave clock sync across a heterogeneous network (LAN servers vs. modem peers). It sits between the game's logical multiplayer state (`rt_net.c`, `rt_game.c`, etc.) and the hardware driver, handling low-level protocol concerns (CRC, node addressing, packet queueing) that the application layer shouldn't see.

## Key Cross-References

### Incoming (callers)
- **rt_net.c**: Calls `ReadPacket()` in main game loop; calls `WritePacket()` to send message packets
- **rt_main.c / rt_game.c**: Calls `InitROTTNET()` during init, `SetTime()` before gameplay starts
- **Message handlers** (elsewhere): Use `WritePacket()` to transmit typed packets (COM_SYNC, COM_START, etc.)

### Outgoing (dependencies)
- **rt_crc.h**: `CalculateCRC()` for packet integrity
- **isr.h**: `ISR_SetTime()` to adjust system clock during sync phases
- **rt_playr.h**: `PlayerInGame()` to check active players, `numplayers` global
- **rt_msg.h**: `AddMessage()` for UI notifications ("All players synched", "Server is synchronizing…")
- **rt_draw.h**: `ThreeDRefresh()` to update screen during long sync waits
- **rt_util.h**: `SafeMalloc()`, `SafeFree()`, `CheckParm()`, `Error()`, `SoftError()`, `AbortCheck()`
- **rt_main.h**: Global `ticcount` (game timer in 70Hz ticks)
- **rottnet.h**: `rottcom_t` structure definition, command/phase constants (CMD_GET, CMD_SEND, SYNC_PHASE\*)

## Design Patterns & Rationale

1. **Real-mode Driver Abstraction**: Encapsulates DOS `int386()` calls and real-mode memory I/O into a thin, command-based API. This isolates the complexity of real-mode/protected-mode transitions to one file.

2. **CRC-Checksum Reliability**: Appends 2-byte CRC to every packet; callers check `badpacket` flag after `ReadPacket()`. Simple but effective for 1990s serial/modem links with bit errors.

3. **Master-Slave Clock Sync Protocol**: A 5-phase handshake that measures round-trip latency and adjusts slave clocks incrementally. Why phased? Allows slaves to react between rounds, accommodates variable network delay, and avoids abrupt clock jumps (which could break gameplay logic).

4. **Role Polymorphism via Globals**: `IsServer`, `networkgame`, `consoleplayer` globals determine whether a player is master or slave. Network games use the `IsServer` flag; modem games always make the initiator master. Node addressing is adjusted per role (`remotenode++` for servers, `remotenode--` for clients).

5. **Guard Flag + One-Shot Init**: `ComStarted` prevents re-initialization; critical because the real-mode driver only needs to be found once.

6. **Synchronous Blocking**: All I/O and sync operations block the entire game. No buffering, no callbacks, no async. This is typical of early 1990s DOS games where the game loop ran single-threaded and couldn't be interrupted.

## Data Flow Through This File

**Initialization**: `CheckParm("net")` → parse command-line address → store in `rottcom` pointer (points into real-mode memory shared with driver).

**Per-Frame I/O**: `ReadPacket()` → call driver with CMD_GET → check `remotenode != -1` → verify CRC → copy to `ROTTpacket` buffer → return true if valid.

**Outgoing**: Application calls `WritePacket(buffer, len, dest)` → appends CRC → writes to `rottcom->data` → sets `rottcom->command = CMD_SEND` and `remotenode` → calls driver → driver transmits asynchronously.

**Sync Flow** (game start):
1. Master calls `SetTime()` → for each remote player, call `SyncTime(i)`
2. `SyncTime()` alternates: master runs `InitialMasterSync()` (announce client to others), then phase loop
3. Master sends SYNC_PHASE0, waits for slave SYNC_PHASE0, advances to PHASE1, sends, waits, etc.
4. Slave receives each phase, calls `SlavePhaseHandler()` to adjust `ticcount`, sends response
5. After phase 5, both agree on a common `ticcount`; latency stored in `transittimes[]`
6. `SetTime()` then broadcasts COM_START, waits for acks, flushes stale packets

## Learning Notes

- **DOS Network Era Pattern**: This is a textbook example of 1990s DOS networking—real-mode drivers, interrupt-based I/O, shared memory buffers, and busy-wait synchronization. Modern engines would use OS-level sockets (Winsock, BSD sockets) and threading.
- **Clock Sync Under Latency**: The phased approach to clock sync is clever: it measures latency in one direction by having slave echo back the master's sent time, allowing the master to compute `delta = (slave_time - master_time + network_latency) / 2` and correct accordingly. This is a variant of network time protocol (NTP) techniques.
- **Trust & Determinism**: The game likely relies on all players' `ticcount` clocks staying synchronized to ensure deterministic, lock-step gameplay (all players execute the same game tick at the same time, given the same inputs).
- **Packet Queueing**: `ReadPacket()` drains one packet at a time; the flushing in `SetTime()` avoids processing old packets. This is a simple FIFO discipline with no prioritization.

## Potential Issues

1. **No Timeout on Sync Hangs**: The busy-wait loops in `SyncTime()` and `InitialMasterSync()` call `AbortCheck()` to allow user abort, but if a remote player goes silent (network outage, crash), the local player's game thread hangs indefinitely.

2. **Inadequate Error Handling on CRC Fail**: A bad packet still returns `true` from `ReadPacket()`; the caller *must* check `badpacket==0` to avoid processing garbage. This is easy to forget and leads to subtle bugs.

3. **Address Translation Complexity**: The `remotenode++` / `remotenode--` logic for server vs. non-server mode is fragile and context-dependent. A mismatch in addressing assumptions would cause packets to be sent to wrong players silently.

4. **No Packet Acknowledgment**: Broadcast packets (e.g., COM_START in SetTime) are sent twice with a busy-wait delay, betting that one will arrive. No actual ACK is expected, so loss could still occur.

5. **Hardcoded Timing Constants**: `SYNCTIME`, `VBLCOUNTER`, phase timings are all compile-time constants; no adaptation to observed latency or network conditions.
