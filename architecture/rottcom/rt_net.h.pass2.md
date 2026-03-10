# rottcom/rt_net.h — Enhanced Analysis

## Architectural Role

This file defines the **network packet protocol schema** for ROTT's multiplayer subsystem. It acts as the contract between the network transport layer (IPX/modem serial drivers) and the game logic layer (`rott/rt_net.c`, which provides `AddClientPacket`, `ControlPlayer`, etc.). Every multiplayer message in the engine must conform to one of these packet types; the GetPacketSize utilities are critical to deserialization during the network receive path.

## Key Cross-References

### Incoming (who depends on this file)
- **`rott/rt_net.c`**: calls `GetPacketSize()` and `GetServerPacketSize()` during packet parsing; creates and sends instances of all COM_*Type structures (evident from `AddClientPacket`, `ControlPlayerObj` references in cross-ref index)
- **`rott/rt_net.h`** (not rottcom): defines higher-level multiplayer orchestration functions (`ControlPlayer`, `ControlRemote`, `ConsoleIsServer`, `CheckForSyncCheck`) that construct and interpret these packet types
- **Game state sync**: `COM_GameMasterType` is broadcast during match initialization to synchronize level, violence settings, player list, and random seed across all nodes

### Outgoing (what this file depends on)
- **`rottnet.h`**: provides constants `MAXPLAYERS`, `MAXNETNODES`, `MAXCODENAMELENGTH` used in player description arrays and game master packet
- **Error system**: `GetPacketSize()` calls `Error()` (fatal halt) on unknown packet types
- **Type definitions**: relies on engine types (`word`, `byte`, `boolean`, `gametype`, `specials`, `battle_type`) defined elsewhere

## Design Patterns & Rationale

**Tagged Union via Packet Type Field**: Every packet struct begins with a `byte type` field (COM_DELTA, COM_TEXT, etc.). `GetPacketSize()` is a **type-driven dispatch** that uses this tag to select the correct struct size. This enables a single void-pointer interface for the transport layer while maintaining type safety at the logical level.

**Variable-Length Aggregation**: `COM_ServerHeaderType` wraps multiple client packets in a single network transmission—`GetServerPacketSize()` walks the data array via pointer arithmetic, calling `GetPacketSize()` on each nested packet. This reflects 1990s optimization: reduce header overhead by batching.

**Delta Compression**: `DemoType` and `MoveType` store only *deltas* (momentum, angle change) rather than absolute position, reducing bandwidth for high-frequency movement updates. This is fundamental to networked games of that era.

**Sound-as-Payload**: `COM_SoundAndDeltaType` aliases `MoveType` and piggybacks sound data on movement packets (the "Remote Ridicule" feature—players taunt each other). The `Sounddata[]` flexible array at the end of `MoveType` makes this work without a separate packet type.

## Data Flow Through This File

```
Transport RX → Raw bytes → GetPacketSize() [determine struct size]
                           → Cast to specific COM_*Type
                           → rt_net.c handler (AddClientPacket, ControlPlayer)
                           
Game Logic → Build COM_*Type struct → rt_net.c sender
                                     → Transport TX → GetServerPacketSize() [batch & send]
```

Critical state-bearing packets:
- **COM_GAMEMASTER**: Broadcast once per match; initializes all players with level/rules
- **COM_DELTA** + **COM_DELTANULL**: High-frequency movement; null variant saves bandwidth when idle
- **COM_SYNCTIME** + **COM_SYNCCHECK**: Synchronize game clocks and validate deterministic state (if `SYNCCHECK==1`)
- **COM_TEXT**: Chat messages, directed to player (255) or team (254)

## Learning Notes

**Era-Specific Network Design**: This is textbook 1990s peer-to-peer gaming:
- No marshalling layer; structs serialize directly to bytes (assumes x86 little-endian)
- No versioning; incompatible game builds cannot network
- Fixed array sizes throughout (no dynamic allocation in packets)
- Conditional compilation for optional features (SYNCCHECK)

**Idiomatic Patterns**:
- "Remote Ridicule" (pre-date of emotes/cosmetics): shows personality in 90s game design
- Deterministic simulation architecture: `COM_SYNCCHECK` validates that all nodes computed identical results, catching desyncs early—a hallmark of lockstep multiplayer engines

**Modern Contrast**: Today's engines use:
- Message serialization libraries (protobuf, msgpack)
- Version fields + backward compatibility
- Delta encoding at the codec layer, not the type system
- Fewer packet types (generic "state update" messages)

## Potential Issues

1. **Unsafe Type Casting**: `GetPacketSize(void * pkt)` casts to `(MoveType *)` to read the type field without bounds checking. A malformed packet could read garbage.

2. **Flexible Array Ambiguity**: `MoveType` ends with `char Sounddata[]`; the size is context-dependent. `COM_SOUNDANDDELTA` size calculation (`sizeof(MoveType) + sizeof(COM_SoundType)`) may not account for padding or the actual sound data length—this relies on careful manual buffer management in rt_net.c.

3. **No Error Recovery in GetServerPacketSize**: If a sub-packet has an invalid type, `GetPacketSize()` halts with `Error()`, leaving the server packet partially parsed. Robust code would return an error code instead.

4. **Version Mismatch Silent Failure**: Two game versions with different packet definitions will desynchronize without clear diagnostic output, making network bugs hard to debug in multiplayer sessions.
