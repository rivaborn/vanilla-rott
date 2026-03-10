# rott/rt_net.h — Enhanced Analysis

## Architectural Role

This file is the **network protocol and command distribution hub** for ROTT's multiplayer system. It sits as an intermediary between the input subsystem (player controls captured in `rt_playr.h`) and the low-level network driver (`rottnet.h`). Critically, it also **embeds the demo recording system** as a first-class concern, creating a tight coupling where networked gameplay and replay are mechanically identical—both use the same `MoveType` packet structure.

The file serves three distinct functions: (1) **protocol specification** with 25+ command types for synchronized game state, (2) **command queueing** via a 256-slot pre-allocated command array for input buffering, and (3) **deterministic demo capture** that enables perfect replay by storing the exact same packet sequence that network players receive.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_playr.h** (player subsystem) calls `ControlPlayer()` to process local input and `UpdateClientControls()` to serialize state
- **rt_main.h** (main game loop) orchestrates frame dispatch via `ProcessServer()`, `ServerLoop()`, and periodic sync checks
- **rt_actor.h** (object/actor system) receives updates via `ControlRemote(objtype * ob)` for remote player state application
- **rt_battl.h** (battle system) receives game configuration via `COM_GameMasterType` packets containing `battle_type options` and `specials`
- Menu/setup code calls `SendPlayerDescription()`, `SendGameDescription()`, `SendGameAck()` during initialization
- Demo subsystem uses `LoadDemo()`, `SaveDemo()`, `DemoExists()`, and `RecordDemo()` for replay functionality

### Outgoing (what this file depends on)
- **rottnet.h** (network driver) provides low-level packet send/receive; defines connection mode constants and player limits
- **rt_actor.h** supplies `objtype` structure for remote player objects that `ControlRemote()` updates
- **rt_playr.h** provides player state structures and the `MAXCODENAMELENGTH` constant
- **rt_battl.h** defines `battle_type` and `specials` types embedded in game master packets
- **develop.h** provides compile-time flags like `SYNCCHECK` that conditionally enable anti-desync features

## Design Patterns & Rationale

### Command Pattern with Pre-allocation
The `CommandType` array (256 slots) and parallel `CommandStatusType` implement a **ring-buffer command queue** without runtime allocation. This guarantees deterministic command sequencing even under packet loss or out-of-order delivery.

**Why pre-allocated?** Deterministic gameplay cannot tolerate malloc failures during critical sync moments. The fixed buffer reflects 1990s real-time constraints.

**Why 256 slots?** Sufficient for LAN/modem speeds of the era; at 30 FPS, ~8 seconds of command history before wraparound.

### VBL-Synchronized Timing
Commands include `int time` fields keyed to `VBLCOUNTER` (vertical blank intervals). The conditional sync rates—`NETSYNCSERVERTIME = VBLCOUNTER` for LAN, `MODEMSYNCSERVERTIME = VBLCOUNTER/4` for modem—show **adaptive protocol tuning** to match latency profiles. LAN syncs every frame; modem every 4 frames to tolerate slower round-trips.

### Demo as Isomorphic Replay
`DemoType` mirrors `MoveType` precisely (time, momx, momy, dangle, buttons). This means: **a recorded demo is literally a serialized network packet stream**. Replay does not require special interpretation; the game loop simply feeds demo packets as if they arrived over the network. This unifies testing, validation, and player-side recording.

### Dual-Protocol Flexibility
Separate flags (`modemgame`, `networkgame`, `standalone`) and conditional packet sizes enable one codebase to handle three network topologies. The file itself is topology-agnostic; rottnet.h abstracts the physical layer.

## Data Flow Through This File

### Player Input → Network
```
Local player presses key
  → ControlPlayer() captures input into player object
  → UpdateClientControls() serializes to MoveType or COM_SoundAndDeltaType
  → AddTextMessage(), AddRemoteRidiculeCommand() queue special packets
  → Commands[] array holds queued packets for transmission to server
```

### Server → Remote Players
```
ProcessServer() receives incoming packets
  → Deserializes MoveType for each remote player
  → ControlRemote(objtype * ob) applies position/angle/buttons to actor
  → Next frame loop renders updated remote player state
```

### Demo Recording Flow
```
RecordDemo() fires each frame (if demorecord=true)
  → Writes DemoType snapshot to demobuffer
  → SaveDemo() flushes buffer to disk when finished
  → LoadDemo() rehydrates disk file into demobuffer
  → Playback loop feeds stored frames back into game loop as MoveType
```

### Game Initialization State Machine
```
Server → COM_GameMasterType (map, violence, player slots, options)
       → Client receives via SetGameDescription()
       ↓
Client → COM_PlayerDescriptionType (character, color, codename)
      → Server receives
      ↓
Client → COM_GameAckType (ready confirmation)
      → Server waits for all clients
      ↓
Server → COM_START
      → All consoles begin frame loop
```

## Learning Notes

### Idiomatic Patterns of the Era
1. **Manual pointer management**: `demobuffer`, `demoptr`, `lastdemoptr` reflect pre-STL C practice; no linked lists or smart pointers.
2. **Fixed packet structures**: Each `COM_*Type` is a single contiguous struct, memcpy'd as-is. No TLV, no field versioning; breaks if struct layout changes.
3. **Synchronous lockstep**: All frames tied to VBLCOUNTER. No frame interpolation, no adaptive timing; latency manifests as visible player jitter, not smooth lag compensation.
4. **Implicit serialization**: Structs sent over the wire without explicit endianness handling; assumes homogeneous x86 platforms.

### Contrast with Modern Engines
- **Modern rollback netcode** (Guilty Gear Strive, SF6) uses unlimited input buffering + deterministic rollback. ROTT's 256-command cap is a precursor but no rollback rewind.
- **Cloud saves & replays** (Overwatch, Valorant) store server-authoritative snapshots. ROTT demos are client-side recordings vulnerable to desync.
- **Protocol versioning** (modern protobuf). ROTT's struct-based protocol has zero forward compatibility.
- **ECS** (modern engines) vs. actor objects. ROTT's tight coupling of input handling to object updates is less composable.

### Core Insight: Determinism as Architecture
This file's entire design enforces **deterministic reproducibility**. The fixed command buffer, VBL-based timing, and isomorphic demo format mean: given identical input sequences and initial state, any two playbacks yield identical results. This is essential for:
- Sync verification (`COM_CheckSyncType`, `CheckForSyncCheck()`)
- Anti-cheat replay validation
- Debugging desyncs (replay with PlayerSync[] logs)

## Potential Issues

1. **Command buffer overflow**: 256-slot limit with no flow control. If a client queues commands faster than the server acks them, early commands are silently overwritten. No warning or retry logic visible.

2. **Endianness vulnerability**: Packet structures (`int time`, `word angle`) sent as raw bytes. Cross-platform play (hypothetical x86↔PowerPC) would silently corrupt all timing and angles.

3. **Sync detection latency**: `CHECKSYNCTIME = (VBLCOUNTER<<2)` only checks sync every 4 frames. Desync that occurs within that window goes undetected; recovery is reactive, not proactive.

4. **Variable-length sound data fragility**: `MoveType.Sounddata[]` is variable-length. Packet parsing depends on exact consistency across all players' sound encoding. A single client with different sound length breaks packet alignment downstream.

5. **Demo buffer unbounded growth**: No visible bounds checking on `demoptr` write position. If a demo records longer than demobuffer capacity, writes corrupt adjacent memory. External code must manage buffer sizing correctly.
