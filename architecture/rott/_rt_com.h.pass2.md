I'll analyze this header file using the provided context and first-pass documentation.

**Note:** The architecture context failed to load, and the cross-reference index provided doesn't include explicit callers of `_rt_com.h`. I'll infer the architectural role from the filename, type signatures, and design patterns visible in the code itself.

---

# rott/_rt_com.h — Enhanced Analysis

## Architectural Role

This private header defines the wire protocol for ROTT's networked multiplayer synchronization layer. It sits at the boundary between the game state simulation and the network transport layer, encapsulating the packet structure and phase constants used during multiplayer initialization and runtime state reconciliation. The `_rt_com` namespace suggests it's part of the core communication subsystem (distinct from higher-level game logic), likely consumed by multiplayer mode initialization (`rt_net.c` or similar) and network drivers.

## Key Cross-References

### Incoming (who depends on this file)
- Likely: `rt_net.c`, `rt_main.c`, or other network initialization code that sets up multiplayer connections
- The private header guard (`_rt_com_private`) suggests selective includes, not public API exposure
- Cross-reference index provided does not explicitly list callers of this header

### Outgoing (what this file depends on)
- **No explicit dependencies**: This is a pure data structure/constant definition header
- Relies on base C types (`byte`, `int`) — assumed defined in `rt_types.h` or similar
- No subsystem calls; only type definitions

## Design Patterns & Rationale

**Multi-Phase Handshake Protocol** (SYNC_PHASE0–SYNC_PHASE5)
- Classic networked game pattern from the 1990s: establishing synchronized game state across peers requires multiple exchange rounds
- Phases likely map to: *Hello/discovery → state transfer → verification → ready* → play
- SYNC_MEMO (value 99) appears to be an out-of-band signal, possibly for periodic state snapshots or debug memoization

**Compact Packet Design**
- 32-byte payload (`SYNCPACKETSIZE`) is tight for mid-90s modems/serial links
- Payload-centric design: metadata (type, phase, timing) + fixed 32-byte data buffer
- Timing metadata (sendtime, deltatime) enables latency compensation and clock skew detection

**Wrapper Struct Pattern**
- `synctype` wraps `syncpackettype` with send/delta timing — separates wire format from transmission context
- Allows retransmission logic and lag prediction without modifying the packet itself

## Data Flow Through This File

1. **Packet Creation** (during multiplayer init): Game code constructs `syncpackettype` with phase identifier and payload
2. **Transmission Context**: Wrapped in `synctype` with timing metadata (when sent, time delta since last sync)
3. **Network Transport**: Packet bytes sent over serial/IPX/whatever transport `rt_net.c` uses
4. **Reception & Reassembly**: Remote peer receives, extracts phase, updates its sync state machine
5. **State Reconciliation**: Once all phases complete, game can proceed with shared state

## Learning Notes

**Idiomatic to 1990s Networked Games:**
- Explicit phase machines replace modern approaches (deterministic lockstep, state trees, or cloud sync)
- Fixed-size packets assume predictable bandwidth; modern engines use variable-length messages and compression
- Send/delta timing suggests latency hiding via client-side prediction, not server authority
- "SYNC_MEMO" phase (value 99) is cryptic — suggests possible snapshot/checkpointing for desync recovery

**Engine Design Philosophy:**
- Synchronization is *structural* — part of the boot sequence, not runtime streaming
- Packet types are enumerated (type field in `syncpackettype`) — likely a dispatch table elsewhere
- The 15-unit `SYNCTIME` constant suggests either frame-based timing (15 ticks ≈ 1/4 second at 60 FPS) or an abstracted time unit

**Modern Contrast:**
- Modern engines use frame-number-based sequencing and delta compression
- RoTT's approach is stateless per-packet; each sync packet is self-contained
- No sequence numbers visible — relies on phase ordering for correctness

## Potential Issues

1. **No Sequence Numbers**: If packets arrive out of order or duplicate on the wire, the phase-only approach may get stuck (depends on transport layer guarantees)
2. **Magic Values**: Phase constants (6, 0, 1, 2, 3, 4, 99) lack semantic clarity — maintainers must cross-reference `rt_net.c` to understand what each phase means
3. **Fixed Payload Size**: 32-byte limit may be insufficient for large game states; no indication of multi-packet reassembly
4. **Clock Skew Handling**: `deltatime` field exists but no visible logic here for handling client/server clock drift

---

**Uncertainty Note:** Without the full architecture context and explicit cross-reference data for `_rt_com.h` callers, some of the "Incoming" dependencies are inferred from naming conventions and design patterns rather than traced calls.
