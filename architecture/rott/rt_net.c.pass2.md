# rott/rt_net.c — Enhanced Analysis

## Architectural Role

This file implements the **network synchronization backbone** of ROTT's multiplayer system, mediating between input polling (rt_util.h, isr.h), game state mutations (rt_actor.h, rt_playr.h), and packet I/O (rt_com.h). It enforces **lockstep deterministic synchronization**: all clients block at `ControlPlayerObj()` waiting for the server's aggregated `COM_SERVER` packet before advancing frame state. This creates a classical turn-based command-replication architecture where commands flow: local input → network → game state application → synchronized update. It also doubles as a **command buffer layer** for demo recording/playback by queuing all input in `PlayerCmds[]` before game physics consume it.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_game.c** (main loop): calls `ControlPlayerObj()` per actor to wait for & apply networked commands
- **rt_main.c** (init): calls `InitializeGameCommands()`, `StartupClientControls()`, `ShutdownGameCommands()`
- **rt_menu.c** (game setup): calls `SetupGameMaster()`, `SetupGamePlayer()` during network handshake; reads `IsServer`, `networkgame`, `modemgame`
- **rt_playr.c** (player AI): reads `PlayerStatus[MAXPLAYERS]` to check if players are alive/left; reads `remoteridicule` flag for sound sync
- **rt_actor.c** (physics): implicitly depends on post-sync `PLAYERSTATE[]` momentum being updated via `UpdatePlayerObj()`
- **rt_draw.c** (rendering): calls `CalcTics()` via timer integration; reads `ticcount` for frame timing

### Outgoing (what this file depends on)
- **rt_com.h** (`rottnet.h`): reads/writes `rottcom` struct (client/server flags, tick step), calls `ReadPacket()`, `WritePacket()`, `BroadcastServerPacket()`
- **isr.h**: calls `CalcTics()`, `ISR_SetTime()` for timer-driven control updates
- **z_zone.h**: calls `SafeMalloc()`, `SafeLevelMalloc()`, `SafeFree()` for command buffer allocation
- **rt_util.h**: calls `PollControls()`, `INL_GetMouseDelta()`, checks global `Keyboard[]`
- **rt_actor.h**, **rt_playr.h**: reads `PLAYERSTATE[]`, calls `MaxSpeedForCharacter()`, reads `characters[]`, modifies actor momentum/buttons
- **rt_sound.h**: calls `SD_Play()`, `SD_GetSoundData()` during `ProcessSoundAndDeltaPacket()` to sync audio cues
- **rt_battl.h**: reads/writes `BATTLE_Team[]`, calls `BATTLE_GetOptions()`, `AssignTeams()` for team-based modes
- **rt_msg.h**: calls `AddMessage()` to display network status (player join, server packets)
- **rt_debug.h**: calls `SoftError()`, `AbortCheck()` for error reporting and development logging
- **Global gamestate**: reads/writes `gamestate` struct (level, difficulty, RNG seed) during setup handshake

## Design Patterns & Rationale

**Lockstep Determinism:**  
All clients wait at `ControlPlayerObj()` for server's aggregated `COM_SERVER` packet before updating actors. This ensures byte-for-byte identical game state across all clients—critical for deterministic physics and fair competitive gameplay. Trades latency (blocks on slowest client) for guarantee of no desyncs.

**Command Queueing (Command Pattern):**  
Input is queued in `PlayerCmds[player][timestamp]` before physics consume it. This decouples input polling from game update, enabling deterministic replay: demo recording just saves the queue, demo playback replays it without user input. `LocalCmds`, `ServerCmds`, `ClientCmds` are separate queues for different roles (client-local, server-aggregated, server-per-client).

**Loss Recovery Protocol:**  
Server resends via `COM_FIXUP` if client requests `COM_REQUEST`. Packets are tagged with timestamp; `LastCommandTime[player]` and `CommandState[i].status` (cs_ready/cs_notarrived/cs_fixing) track which packets are in-flight or missing. Avoids both stalling on loss and desync on silent drops.

**Time-Sync Soft Handshake:**  
Clients advance `controlupdatetime` independently; server broadcasts its `serverupdatetime`. Clients apply commands when their local time ≥ packet timestamp, allowing natural clock drift correction without explicit resync interrupts.

**Dual-Purpose Command Buffers:**  
The same `MoveType` packet structure used for network transport is also logged to disk during demo recording. Playback reads from disk instead of input devices, replaying exact command sequence. Enables "watch replay" and regression testing.

## Data Flow Through This File

```
Input Polling (per-frame):
  UpdateClientControls() 
    ├─ PollControls() → controlbuf[], buttonbits
    ├─ Create MoveType { time, dmomx, dmomy, angle, buttons }
    ├─ PrepareLocalPacket() → LocalCmds[time]
    ├─ WritePacket() → network (or demobuffer if demorecord)
    └─ controlupdatetime += controlsynctime

Network Dispatch (incoming packet):
  CheckForPacket() → ProcessPacket(pkt, src)
    ├─ if COM_DELTA/SOUNDANDDELTA: AddClientPacket()
    │   └─ PlayerCmds[src][pkt.time] = pkt
    ├─ if COM_SERVER: AddServerPacket()
    │   └─ ServerCmds aggregates all players' deltas
    ├─ if COM_REQUEST: ResendLocalPackets()
    │   └─ Resend via COM_FIXUP
    └─ if COM_SYNC: Update client's time offset

Server Aggregation (only if IsServer):
  ProcessServer()
    ├─ Wait: AreClientsReady(){ for all i: CommandState[i].status==cs_ready }
    ├─ SendFullServerPacket()
    │   └─ Gather ClientCmds[i] → COM_ServerHeaderType packet
    │   └─ BroadcastServerPacket() to all clients
    └─ Repeat next frame

Game State Application (per player, per frame):
  ControlPlayerObj(player)
    ├─ Wait: CommandState[player].status == cs_ready (timeout NETWORKTIMEOUT)
    ├─ ProcessPlayerCommand()
    │   ├─ UpdatePlayerObj(player) 
    │   │   ├─ Unpack MoveType → PLAYERSTATE[player].dmomx/dmomy/angle
    │   │   └─ Apply button state: Keyboard[], buttonstate[]
    │   └─ ApplyPlayerCommand() → actor physics simulation
    └─ Advance CommandState[player].index for next frame
```

## Learning Notes

**Lockstep Synchronization (Mid-90s Standard):**  
This is textbook turn-based deterministic multiplayer from Apogee's era. Every frame, all clients stall until the server broadcast arrives. Modern engines use **client-side prediction + server reconciliation** (optimistic, lower latency) or **rollback netcode** (rollback on misprediction). But lockstep is simpler to reason about and deterministic for authoritative server gameplay.

**Command Pattern in Multiplayer:**  
Commands are immutable, timestamped actions queued per-player. This is the foundation of deterministic replay—identical command sequences produce identical results. The engine logs commands to disk for demo recording; modern engines use this for networking, replays, and input recording simultaneously.

**Interrupt-Driven Timing (DOS Legacy):**  
`CalcTics()` and `ISR_SetTime()` are interrupt-service-routine calls. The timing model is **tic-based** (discrete time steps, ~20ms), not frame-independent. This is vastly simpler than continuous timing and ensures deterministic behavior across all clients. Modern engines abstract timing to allow variable framerates.

**Global State Coupling:**  
Game state lives in globals (`gamestate`, `PLAYERSTATE[]`, `rottcom`), not encapsulated. This makes the code harder to test and refactor, but allowed aggressive optimization in 1995. A modern engine would use a scene graph or ECS to isolate networked entity state from networking code.

## Potential Issues

1. **Lockstep Stalling (Latency):**  
   `ProcessServer()` blocks until all clients ready (line ~1800+). If one client lags, the entire server waits. On slow connections, this cascades into noticeable frame stuttering. Threshold: `NETWORKTIMEOUT` causes timeout-based fallback, but risks desync.

2. **Time Sync Race Condition:**  
   `controlupdatetime` is incremented in `UpdateClientControls()` while `CommandState[i].status` is checked asynchronously in `ControlPlayerObj()`. If a packet arrives between the increment and the check, timestamp ordering could be violated. No explicit mutual exclusion observed (but DOS single-threaded execution masks this).

3. **Memory Leak on Early Exit:**  
   If `InitializeGameCommands()` succeeds but game aborts before `ShutdownGameCommands()` (e.g., network error during handshake), allocated buffers in `PlayerCmds[]`, `LocalCmds`, `ServerCmds` are not freed. No cleanup handlers or RAII-like patterns visible.

4. **Demo Record/Playback Entangled:**  
   `demorecord`, `demoplayback`, `demobuffer` live in main game loop, not abstracted. Recording and playback paths are interleaved with network code, making both hard to reason about independently. A command-replay subsystem would be cleaner.

5. **Silent Packet Loss (No ACK):**  
   Server broadcasts but doesn't track whether clients received it. If a broadcast is lost and the client doesn't `COM_REQUEST` in time, the client is permanently desynced. Relies on loss detection timeout, not proactive ACK.
