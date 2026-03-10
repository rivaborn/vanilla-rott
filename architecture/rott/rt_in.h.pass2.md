# rott/rt_in.h — Enhanced Analysis

## Architectural Role

This is the **input abstraction layer** for a DOS-era multiplayer game engine. It sits at the foundation of the player control pipeline, translating heterogeneous hardware (keyboard, mouse, joystick, modem) into a unified **ControlInfo** structure. The system supports both single-player (local control devices) and networked multiplayer (ModemMessage integration), making it a critical junction between player input, game logic (rt_playr, game loop), and network synchronization (rottnet).

## Key Cross-References

### Incoming (Callers)
- **Game Loop / Main (rt_main):** Calls `IN_Startup` / `IN_Shutdown` for init/cleanup; likely polls `IN_ReadControl` per frame
- **Player Controller (rt_playr):** Depends on `IN_ReadControl()` to fetch normalized `ControlInfo` for each player's movement and actions
- **Menu System (rt_menu):** Uses `IN_WaitForKey()`, `IN_WaitForASCII()`, `QueueLetterInput()` for player name entry and menu navigation
- **Network Code (rt_net):** Consumes **ModemMessage** struct (MSG global) to handle networked messages; synchronizes remote player input via modem/network
- **Debug/Cheat Systems:** May directly read input globals (LastScan, LastASCII, LetterQueue) for cheat codes or debug commands

### Outgoing (Dependencies)
- **develop.h:** Compile-time configuration flags (determines feature availability—shareware vs. full, cheats, etc.)
- **rottnet.h:** Network constants (`MAXPLAYERS`, `MAXNETNODES`) and likely modem/network message definitions
- **Hardware Layer (implicit):** Low-level joystick/mouse/keyboard ISR or polling routines (INL_* functions implemented in RT_IN.C)

## Design Patterns & Rationale

### 1. **Hardware Abstraction Layer (HAL)**
- **ControlType enum** (keyboard/joystick/mouse variants) allows runtime selection of input device per player
- **ControlInfo struct** provides a unified interface: buttons, x/y position, motion, direction—regardless of source
- **Rationale:** Supports game configuration (players choose control device) without game logic changes; enables local multiplayer with mixed devices (one player keyboard, another joystick)

### 2. **Interrupt-Driven Keyboard**
- **volatile int LastScan** suggests keyboard ISR updates this; main thread reads it
- **IN_ClearKeysDown()** implies tracking of key state across frames
- **Rationale:** DOS era standard for responsive input without polling overhead

### 3. **Device Calibration Pattern**
- **JoystickDef** stores per-joystick min/max/threshold values and X/Y multipliers
- **IN_SetupJoy()** and **INL_SetJoyScale()** allow runtime calibration
- **Rationale:** DOS joysticks had wide hardware variance; calibration was essential for playability across systems

### 4. **Layered Initialization**
- **IN_Startup()** → INL_Start* (low-level) + IN_Default() → configure per-player
- **Rationale:** Separates hardware detection from user configuration; supports graceful fallback (e.g., no mouse → keyboard-only)

### 5. **Modular Text Entry**
- **LetterQueue[] + LastLetter + QueueLetterInput()** isolates text input logic
- **Rationale:** Reusable for player name entry, chat/modem messages, or configuration input

## Data Flow Through This File

```
┌─────────────────────────────────────┐
│  Hardware (Keyboard, Mouse, Joy)   │
│  └─> ISR / Polling Layer           │
└──────────────┬──────────────────────┘
               │
        ┌──────▼───────────────────┐
        │  Global State            │
        │  - LastScan              │
        │  - Joy_x, Joy_y          │
        │  - MousePresent, etc.    │
        └──────┬────────────────────┘
               │
        ┌──────▼────────────────────────────┐
        │  IN_ReadControl(player, &info)   │
        │  - Reads Controls[player] device │
        │  - Queries device-specific state │
        │  - Computes direction from axes  │
        │  - Outputs ControlInfo           │
        └──────┬────────────────────────────┘
               │
   ┌───────────┴────────────┬──────────────┐
   │                        │              │
   ▼                        ▼              ▼
rt_playr                rt_menu        rt_net
(Movement/Action)    (Navigation)  (Sync input)
```

**Text Entry Path:**
- `QueueLetterInput()` → LetterQueue[] (+ LastLetter index) → Menu/Config reads LetterQueue[0..LastLetter-1]

**Networking Path:**
- Modem layer populates **MSG** (ModemMessage) → rt_net reads MSG.towho, .directed, .textnum for message routing/display

## Learning Notes

### Era-Specific Idioms
1. **Volatile for ISR-shared state:** `volatile int LastScan` is classic DOS interrupt-driven design (ISR modifies, main thread reads without explicit synchronization).
2. **Device multiplexing at init time:** No dynamic hot-plug support; devices detected once at startup. Reflects 1990s hardware constraints.
3. **Explicit calibration loops:** Joystick thresholds and multipliers required manual tuning—no autodetect/profiling. Players had to run calibration.
4. **Modem play at the I/O layer:** Most games handled modem sync above input; this engine integrates it lower, suggesting tight coupling between input and network.

### Modern Engine Differences
- **Modern:** Event-driven input (polling on-demand, callbacks) vs. **ROTT:** Global state + frame-based polling (IN_ReadControl once per frame)
- **Modern:** Game objects implement controller interface vs. **ROTT:** Centralized ControlInfo translation
- **Modern:** Hot-plug device support vs. **ROTT:** Static detection at startup
- **Modern:** Separated input, logic, and networking vs. **ROTT:** ModemMessage struct couples message I/O to input layer

### Architectural Insights
- **Input as a bridge to multiplayer:** The presence of ModemMessage and Controls[MAXPLAYERS] shows that network play was a core design goal from the start, not retrofitted.
- **Player abstraction:** ControlInfo struct + per-player ControlType allows the same player control logic (rt_playr) to work for local or remote players with minimal branching.
- **Hardware-agnostic game loop:** Game logic (rt_playr, rt_menu) need not know *which* device is active—just read ControlInfo.

## Potential Issues

1. **No bounds checking on LetterQueue:** `QueueLetterInput()` is not visible, but if it doesn't verify `LastLetter < MAXLETTERS`, buffer overflow is possible during name entry.
2. **Race condition on shared globals:** ISR writes `LastScan` while main thread reads it via `IN_WaitForKey()` or polling. No explicit synchronization; relies on atomic word access (safe on x86, but fragile if ported).
3. **ModemMessage state management:** No visible lifecycle docs—unclear who initializes MSG or when it's cleared. Could lead to stale message data if not reset properly between rounds.
4. **No input buffering for menu:** Text entry uses a queue, but `IN_WaitForKey()` blocks; if called during networking, input latency could cause desync in multiplayer.

---
