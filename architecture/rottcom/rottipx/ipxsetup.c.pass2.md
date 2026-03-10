# rottcom/rottipx/ipxsetup.c — Enhanced Analysis

## Architectural Role

**ipxsetup.c** is the multiplayer initialization orchestrator for IPX-based networked ROTT games. It bootstraps the entire networking subsystem by discovering peers, negotiating player assignments, and blocking until all nodes are ready before launch. This file acts as a gateway between the user's command-line options and the low-level IPX driver (via `ipxnet.h`), absorbing the complexity of network handshaking so the game engine (`rottnet.h` / `LaunchROTT()`) can assume a stable, initialized multiplayer state.

## Key Cross-References

### Incoming (who depends on this file)
- **Game launcher** → Executes `main()` with `-server`, `-nodes`, `-socket` flags
- **IPX driver** → Hooks `NetISR()` as an interrupt handler for async packet events
- No other `.c` files directly call functions from ipxsetup.c (this is a standalone program entry point)

### Outgoing (what this file depends on)
- **rottcom/rottipx/global.c** → `CheckParm()` for command-line parsing
- **rottcom/rottipx/ipxnet.c** → `InitNetwork()`, `ShutdownNetwork()`, `SendPacket()`, `GetPacket()`, `AllocatePackets()`
- **rottnet.h** (game engine) → `LaunchROTT()`, `ShutdownROTTCOM()`, `Error()` (error signaling)
- **rottcom shared struct** → `rottcom` global (communication block), `nodeadr[]` array (node routing table), `remotetime`, `remoteadr`
- **DOS/BIOS** → `bioskey()` for keyboard abort, `gettime()` for timeout, `randomize()` / `random()` for message selection

## Design Patterns & Rationale

**1. State Machine (per-client tracking)**
- Each connected node transitions: `client_NoResponse` → `client_Echoed` → `client_Done`
- Prevents duplicate player assignments and detects hung clients
- Idiomatic for 1990s DOS networking where acknowledgment-based handshakes were essential

**2. Polling with Timeout**
- Main loop calls `GetPacket()` in a tight loop, checking wall-clock seconds
- Every 10 seconds (`MAXWAIT`), broadcasts a "find client" probe + humorous message
- Tradeoff: Avoids blocking ISR but consumes CPU; modern engines would use event-driven or async I/O

**3. Client/Server Symmetry**
- Both modes run the same `LookForNodes()` loop but take different branches
- Server publishes player slots; clients respond and await assignment
- Supports delayed master client joins via `masterset` flag (resets numbering if master arrives late)

**4. Message Cycling (No Repeats)**
- `StringUsed[]` bitmap ensures each of 25 humorous messages appears exactly once per cycle
- `GetRandomMessage()` resets when exhausted—charming UX pattern from pre-internet era
- Modern engines would simply show a progress bar

**5. ISR Decoupling**
- `NetISR()` is hooked by the IPX driver and runs asynchronously
- Only checks `rottcom.command` flag (send vs. receive); delegates work to main thread
- Minimizes ISR complexity—actual packet parsing happens in `LookForNodes()`'s `GetPacket()` loop

## Data Flow Through This File

```
User launches: game.exe -server -nodes 4 -socket 0x882a
    ↓
main() → CheckParm() [parse flags]
    ↓
InitNetwork() → allocate IPX socket, hook NetISR
    ↓
ResetNodeAddresses() → zero state, mark server as player 1
    ↓
LookForNodes() ─→ [polling loop, 10-sec timeout]
    │
    ├─ Server: broadcast cmd_FindClient → recv cmd_HereIAm → send cmd_YouAre → recv cmd_IAm
    │
    └─ Client: recv cmd_FindClient → send cmd_HereIAm → recv cmd_YouAre → send cmd_IAm
    ↓
[all players confirmed in clientstate[]]
    ↓
rottcom.numplayers, rottcom.consoleplayer set
    ↓
LaunchROTT() → game engine starts with stable peer table
    ↓
Shutdown() → ShutdownROTTCOM(), ShutdownNetwork()
```

Packet types (command field):
- **cmd_FindClient(1)**: Server probes for clients
- **cmd_HereIAm(2)**: Client responds to probe
- **cmd_YouAre(3)**: Server assigns player number
- **cmd_IAm(4)**: Client confirms identity
- **cmd_Info(6)**: Server broadcasts status/humorous messages
- **cmd_AllDone(5)**: Server signals game ready

## Learning Notes

**Idiomatic 1994–1995 DOS / IPX networking patterns:**
1. **ISRs for I/O**: Network events arrive asynchronously via hardware interrupts; main thread polls for readiness.
2. **Global state for communication**: `rottcom`, `nodeadr[]`, `clientstate[]` are shared mutable globals—no TLS, no async queues.
3. **Blocking setup phase**: Network discovery is synchronous; game cannot start until all nodes confirm. Modern engines decouple setup from gameplay (e.g., matchmaking servers, late joins).
4. **Direct BIOS calls**: `bioskey()` for escape-key abort, `gettime()` for timeouts. Modern engines use OS event loops.
5. **Humorous feedback**: Waiting messages are a charm artifact; modern engines show deterministic progress meters or spinners.

**Contrasts with modern engines:**
- No connection retry logic, no graceful degradation, no spectator mode for failed peers
- Synchronous state machine (all-or-nothing) vs. modern async scene loading or player drop-in
- No server election algorithm (master flag is set before handshake, not negotiated)
- All setup happens in one function; modern engines would split discovery, negotiation, and resource loading

## Potential Issues

1. **No bounds check in `PrintRandomMessage()`**: If `message` index >= `MAXNETMESSAGES`, `NetStrings[message]` accesses out of bounds. Mitigated by `GetRandomMessage()` returning valid indices, but a defensive check would help.

2. **Infinite wait if master client never appears**: If `-master` flag is set but master doesn't broadcast `extra==1`, server resets numbering and may deadlock awaiting the master. No explicit timeout or error for this case.

3. **No peer keep-alive during game**: Setup phase ensures all peers are reachable, but if a peer drops *during* gameplay, no ping/heartbeat mechanism is shown here. Game loop may hang waiting for packets from dead peers.

4. **Global state fragility**: All node discovery state lives in file-static and global variables. If `LookForNodes()` is called again mid-game, `nodeadr[]` and `clientstate[]` could be reset, breaking existing connections.

5. **Assumes single game instance**: No locking or mutual exclusion around `rottcom`. If two setup programs run concurrently on the same node, they'll corrupt each other's state.
