# rott/rt_com.h — Enhanced Analysis

## Architectural Role

`rt_com.h` defines the **lowest-level network abstraction layer** in ROTT's multiplayer infrastructure. It sits at the platform boundary, sandwiched between the game logic (`rt_net.h`) and platform-specific drivers (modem serial code in `rottcom/rottser/`, IPX code in `rottcom/rottipx/`). This header abstracts away whether packets arrive via serial modem, IPX LAN, or other transports—the game sees only `ReadPacket()`/`WritePacket()` and a shared `ROTTpacket` buffer.

## Key Cross-References

### Incoming (who depends on this file)
- **`rt_net.c` / `rt_net.h`** — High-level networking layer. Functions like `AddClientPacket`, `ControlPlayerObj`, `CheckForSyncCheck`, `ConsoleIsServer` build game logic on top of these primitives. Likely calls `ReadPacket()` in main loop and `WritePacket()` to broadcast state.
- **Game main loop** — Calls `InitROTTNET()` at startup and `SetTime()` each frame for tick synchronization.

### Outgoing (what this file depends on)
- **`rottnet.h`** — Provides platform detection macros (`__WATCOMC__`), buffer size constants (`MAXCOMBUFFERSIZE`), and the `rottcom_t` structure (player/node state). Acts as the **interface to platform drivers** (serial, IPX, etc.).
- **Platform drivers** — Implicitly: `rottcom/rottser/` (modem) and `rottcom/rottipx/` (IPX) implement the actual I/O; this header just declares the game-facing API.

## Design Patterns & Rationale

**Layered abstraction with platform indirection:**
- The header declares only function signatures, not implementations—allows multiple driver implementations (modem/IPX/TCP) to be swapped without recompiling game code.
- Shared global `ROTTpacket` buffer reduces per-frame allocation overhead (important on DOS).
- Simple polling interface (`ReadPacket` returns boolean) fits 90s cooperative multitasking; no callbacks or async I/O.

**Tradeoff:** Tight coupling via global `ROTTpacket` and `consoleplayer` — modern engines would use callback/event systems instead. But this works well for a single-threaded DOS game.

## Data Flow Through This File

```
Network Driver
    ↓
ReadPacket() → ROTTpacket[] (shared buffer) → rt_net.c reads & routes to players
                consoleplayer field identifies sender
                
Game state in rt_net.c
    ↓
WritePacket(buffer, len, destination) → Network Driver
                                      → Serial/IPX transport
                                      
SetTime() / GetTransitTime()
    ↓
controlsynctime (ticked by game loop)
    ↓ (used by rt_net.c for frame synchronization)
```

## Learning Notes

**Idiomatic to 90s game networking:**
- **Polling model, not event-driven** — Modern engines use queues, callbacks, or async/await.
- **Global state heavy** — `consoleplayer`, `ROTTpacket`, `badpacket` are file-scoped globals; no per-connection structs. Works for 2–4 player games but doesn't scale.
- **Simple latency handling** — `GetTransitTime()` exposes round-trip estimates for lag compensation; no buffer management visible at this layer.
- **Monolithic buffer** — A single `ROTTpacket[MAXCOMBUFFERSIZE]` is reused each frame. Modern engines use ring buffers or per-packet allocation.

**Connection to engine concepts:**
- This is the **I/O subsystem boundary** in a classic engine stack.
- Equivalent to a networking "device driver" that abstracts platform differences.
- The `controlsynctime` global is a **game tick / frame counter**, used for deterministic replay and network frame locking (common in peer-to-peer networked games of this era).

## Potential Issues

- **No error handling exposed** — `ReadPacket()` returns boolean but no way to distinguish timeout, corrupt packet, or disconnect. Callers in `rt_net.c` must infer state from `badpacket` flag.
- **Race condition risk** — Global `ROTTpacket` buffer shared without apparent synchronization; if main loop and an ISR (interrupt) both access it, data corruption is possible (though less likely in cooperative DOS).
- **Incomplete API** — No way to initiate disconnection or query connection status; suggests connection setup/teardown is managed elsewhere (likely in `rottnet.h` or platform driver).

---

**Note:** Architecture context failed to load (max turns). Cross-reference analysis is limited to the provided excerpt. Full subsystem map would clarify which other modules depend on these functions.
