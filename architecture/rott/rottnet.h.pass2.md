# rott/rottnet.h — Enhanced Analysis

## Architectural Role

`rottnet.h` defines the **inter-process communication (IPC) boundary** between the ROTT game engine and a separate network driver process (ROTTCOM). It establishes the shared memory protocol (`rottcom_t`), interrupt signaling scheme, and process lifecycle functions that decouple the game from hardware/transport details. This is a classic 1990s driver architecture: the driver runs as an independent executable (configured via `rottcom_t`, launched via `LaunchROTT()`), while the game reads/writes to shared memory and triggers interrupts to signal events.

## Key Cross-References

### Incoming (who depends on this file)
- **`rt_net.c` / `rt_net.h`**: Game-side networking subsystem (player sync, packet queueing, client/server logic)
  - Functions like `AddClientPacket`, `ControlPlayer`, `ConsoleIsServer` likely read/write via `rottcom` global
  - Uses `rottcom_t` fields like `consoleplayer`, `numplayers`, `gametype` to manage session state
- **`rt_main.c`**: Game initialization/shutdown
  - Calls `ShutdownROTTCOM()` during cleanup
  - Likely calls `LaunchROTT()` during multiplayer setup
- **ROTTCOM driver process**: Reads/writes shared `rottcom` struct; implements `NetISR` handler

### Outgoing (what this file depends on)
- **`global.h` / `develop.h`**: Compiler/build config, basic types (`boolean`, `short`, `char`), I/O macros (`outp`, `inp`)
- **Legacy x86 I/O ports** (0x3c8, 0x3c9): VGA palette registers (used by `I_ColorBlack` macro)
- **Interrupt vector mechanism**: Relies on OS/compiler support for interrupt hooks (details in BIOS/DOS/runtime)

## Design Patterns & Rationale

1. **Shared Memory IPC**: Pre-allocates `rottcom_t` as global or pointer; driver and game exchange data synchronously via interrupt signals. This avoids the overhead of sockets/pipes and provides tight timing guarantees for multiplayer sync.

2. **Interrupt-Driven Signaling**: Rather than polling, the driver signals game via interrupt (`intnum`), triggering `NetISR()`. This was efficient on 1990s hardware with limited CPU cycles.

3. **Conditional Struct Layout** (`#pragma pack (1)` in Watcom): Ensures binary layout matches between game and driver; critical for IPC where pointer sizes and alignment differ between contexts.

4. **Compiler Abstraction**: `#if __WATCOMC__` wraps Watcom-specific pragmas and pointer vs. direct reference for `rottcom`. This allowed the codebase to support both Watcom (development) and non-Watcom (retail build) toolchains.

5. **Product Variant Abstraction**: `#if SHAREWARE == 1` adjusts max player count (5 vs. 11), suggesting a single codebase could build both shareware demo and full product without duplication.

6. **Command Pattern**: `rottcom_t.command` field (CMD_SEND, CMD_GET, etc.) decouples game logic from driver implementation—game doesn't need to know how packets traverse modem or network.

## Data Flow Through This File

```
Game Process                           Driver Process
─────────────────────────────────────────────────────
1. Game populates rottcom_t
   ├─ command ← CMD_SEND
   ├─ data[] ← outgoing packet
   ├─ datalength ← bytes
   └─ remotenode ← dest
2. Game triggers interrupt (intnum)
                  ────────→  Driver IRQ handler
3. Driver reads rottcom_t
                              ├─ Encapsulates packet
                              ├─ Sends via modem/LAN
4. Driver writes rottcom_t
   ├─ command ← CMD_GET
   ├─ data[] ← incoming packet
   └─ datalength ← bytes read
5. Driver triggers interrupt
         ←────  Game NetISR()
6. Game reads rottcom_t, processes packet
   └─ Updates game state (player positions, etc.)
```

**Key state**: `consoleplayer`, `numplayers`, `gametype`, `ticstep` are configured once at session start; `client` flag distinguishes server-side latency hiding vs. client input buffering.

## Learning Notes

1. **1990s Multiplayer Architecture**: This IPC pattern was common before UDP/TCP were practical for real-time games (latency, packetization). Modern engines use sockets + async I/O, but the concepts persist: shared state, interrupt-like events, command queueing.

2. **Shareware vs. Full Split**: Player count limits embedded in headers show how Apogee built one codebase for multiple product tiers—the shareware still supports online play (5 players), just limited capacity.

3. **Modem as First-Class Transport**: The `MODEM_GAME` / `NETWORK_GAME` abstraction suggests modem/serial play was equally supported to LAN, important context for 1995. The driver likely handled both transparently.

4. **Interrupt Vector Management**: `GetVector()` retrieves the hooked ISR address, likely used by driver to restore old handler on shutdown. This is defensive programming for DOS/early Windows where improper interrupt cleanup could crash the system.

5. **Tic-Step Abstraction**: `ticstep` field (every 1 or 2 tics) hints at dynamic bandwidth adaptation—if network is overloaded, game can reduce sync frequency. Modern engines do similar via variable tick rates.

6. **Palette I/O in Networking Header**: The `I_ColorBlack` macro seems out of place in a network header but suggests the driver might render debug UI or tunnel video output. This is typical of 1990s driver tooling.

## Potential Issues

1. **No Error Handling**: Functions like `ShutdownROTTCOM()` return `void`; there's no way for game to detect if driver failed. Robust multiplayer would want shutdown status or timeout handling.

2. **Fixed Buffer Sizes**: `MAXPACKETSIZE = 2048` is hardcoded. If packets exceed this, the game will silently overflow `rottcom_t.data[]`. No bounds checking is visible in this header.

3. **Race Conditions**: The header doesn't document synchronization (mutexes, atomic flags). If game and driver both write `rottcom_t` simultaneously, corruption can occur. Likely relied on interrupt masking on single-threaded DOS.

4. **Platform-Specific Interrupt Setup**: `intnum` and `GetVector()` assume x86 DOS/early Windows interrupt architecture. Porting this code to modern OSes (Linux, Win32) would require complete redesign.

5. **Missing Validation**: No version/magic number in `rottcom_t` (though `ROTTCOM_ID` constant exists—unclear if used). Mismatched game/driver versions could silently corrupt state.
