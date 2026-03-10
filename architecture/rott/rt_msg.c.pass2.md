# rott/rt_msg.c — Enhanced Analysis

## Architectural Role

**rt_msg.c** is the engine's unified on-screen messaging subsystem, handling both transient gameplay notifications (pickups, cheats, system alerts) and persistent UI overlays (multiplayer chat, player selection menus). It acts as a bridge between game logic and the rendering pipeline, integrating closely with the multiplayer networking layer (rt_net.c) for directed chat and the video subsystem (rt_vid.c) for rendering. The system is driven per-frame by the game loop via `DrawMessages()` and `UpdateMessages()`, and receives event-driven additions via `AddMessage()` from scattered call sites across the engine.

## Key Cross-References

### Incoming (who depends on this file)

- **Game events** → `AddMessage()`: Called from throughout the codebase when pickups occur, cheats are triggered, system events fire (likely in rt_playr.c, rt_actor.c, rt_game.c, rt_debug.c)
- **Render pipeline** → `DrawMessages()`, `UpdateMessages()`: Called per-frame from rt_draw.c or rt_main.c game loop
- **Multiplayer chat input** → `UpdateModemMessage()`, `ModemMessageDeleteChar()`, `FinishModemMessage()`: Called from input handlers during modem message composition (likely rt_net.c or rt_com.c)
- **Game state machine** → `InitializeMessages()`, `ResetMessageTime()`: Called during level/game initialization and state transitions

### Outgoing (what this file depends on)

- **Multiplayer subsystem** (rt_net.c/rt_net.h): 
  - Calls `AddRemoteRidiculeCommand()` to route taunt messages to specific players
  - Calls `AddTextMessage()` to broadcast chat or directed messages to network
  - Reads `numplayers`, `consoleplayer`, `PLAYERSTATE[]`, `gamestate.teamplay` for multiplayer routing
  - Uses `MSG` global struct from rt_com.h (length, directed flag, recipient)

- **Rendering subsystem** (rt_vid.c/rt_vid.h):
  - Calls `DrawIString()` to render message text with color
  - Calls `DrawTiledRegion()` and `DrawCPUJape()` to restore background during soft-erase
  - Reads `fontcolor` global to control text color

- **Display/viewport** (rt_view.h):
  - Calls `SHOW_TOP_STATUS_BAR()` to adjust message Y-position
  - Reads `YOURCPUSUCKS_Y`, `egacolor[]` for positioning and palette lookups

- **Resource loading** (w_wad.c):
  - Calls `W_CacheLumpName()` to load font sprite (`"ifnt"`) and background tile (`"backtile"`) per-frame (cached by engine)

- **Memory subsystem** (z_zone.h):
  - Calls `SafeMalloc()` / `SafeFree()` for message text buffers

- **Game state** (rt_main.h):
  - Reads `GamePaused` flag (skips updates if true)
  - Reads `ticcount` for timer deltas and automatic message expiration

## Design Patterns & Rationale

**1. Fixed Object Pool with Priority Queue:**
- `Messages[]` is a fixed array; `AddMessage()` calls `GetFreeMessage()` which either finds an inactive slot or evicts the message with the smallest `tictime` (oldest temporary message)
- `MessageOrder[]` is rebuilt after each add/delete via `GetMessageOrder()`, sorting by `tictime`
- **Rationale**: Avoids repeated malloc/free in a real-time engine; predictable memory footprint; sorting on demand is acceptable for `MAXMSGS` (likely ~16–32)

**2. Permanent vs. Temporary Distinction:**
- Temporary messages: allocated with `MESSAGETIME` countdown, auto-expired each frame by `UpdateMessages()`
- Permanent messages (menus, chat): allocated with `COM_MAXTEXTSTRINGLENGTH` (fixed buffer) and negative `tictime` to sort first
- **Rationale**: Avoids leaking memory from persistent UI; negative tictime ensures menus stay on-screen and above temporary notifications

**3. Soft Erase with Amortized Restoration:**
- `DisplayMessage()` doesn't clear background; instead `DeleteMessage()` increments `UpdateMessageBackground` and sets `EraseMessage[]` flags
- `RestoreMessageBackground()` iterates once per frame, redrawing tiles and status bar only for messages marked with `EraseMessage[] > 0`, decrementing counters over 3 frames
- **Rationale**: Reduces per-frame rendering cost and visual flicker; overlaps tile redraw with other frame work

**4. Type-Based Filtering & Color Routing:**
- Each message type (MSG_CHEAT, MSG_SYSTEM, MSG_GAME, etc.) has a dedicated case in `DisplayMessage()` determining color, prefix, and whether to suppress if `!MessagesEnabled`
- Remote/modem messages get special prefix handling for clarity in multiplayer
- **Rationale**: Easy to add new message types; clear visual distinction helps players distinguish urgent (red system) from informational (green game) messages

**5. Directed Messaging with Player Selection Menu:**
- `FinishModemMessage()` detects `MSG.directed` flag; if set, calls `DrawPlayerSelectionMenu()` which creates temporary permanent messages listing players 0–9 and team/all options
- User selects a number, which routes the message via `AddRemoteRidiculeCommand()` or `AddTextMessage()`
- **Rationale**: Separates chat composition (rt_com.c) from routing (rt_net.c), centralizing multiplayer UI flow

## Data Flow Through This File

**Event-Driven (Game Logic):**
```
Game event (pickup, cheat)
  ↓
AddMessage(text, flags)
  → GetFreeMessage() [evicts old if needed]
  → SetMessage(num, text, flags) [allocates buffer, sets tictime]
  → GetMessageOrder() [re-sorts MessageOrder[] by tictime]
  → Returns message index
```

**Per-Frame Rendering:**
```
Game loop calls DrawMessages()
  → W_CacheLumpName("ifnt") [load color font]
  → FOR i in MessageOrder[0..TotalMessages):
       DisplayMessage(i, position) [render with color, prefix]
  → UpdateMessages() [decrement tictime, delete if <= 0]

After rendering (next frame?):
  → RestoreMessageBackground() [redraw tiles for erased regions]
```

**Multiplayer Chat:**
```
Input handler calls UpdateModemMessage(num, c)
  → Appends char to Messages[num].text, adds cursor underscore
  → Increments UpdateMessageBackground

User presses Enter:
  → FinishModemMessage(num, true)
    → If directed: DrawPlayerSelectionMenu() [add temp menu messages]
    → User selects number
    → AddRemoteRidiculeCommand() or AddTextMessage() [sends to rt_net.c]
    → DeleteMessage(num) [remove chat composition message]
```

## Learning Notes

**Idiomatic Patterns from 1990s Game Engines:**
1. **Fixed-size arrays + sorting** instead of dynamic lists (simpler, predictable perf)
2. **Type-based dispatch** via switch on flags (vs. virtual methods or callbacks)
3. **Amortized screen restoration** (3-frame soft erase) to reduce per-frame cost
4. **Global flag toggles** (MessagesEnabled, GamePaused) for high-level control without callback chains

**Modern Engine Contrasts:**
- Modern engines often use **ECS** (Entity-Component-System) where UI is data-driven; rt_msg.c uses imperative fixed arrays
- Modern engines defer **screen clearing** to a dedicated clear pass; rt_msg.c explicitly restores tiles to avoid black flash
- Modern engines separate **input handling** from **rendering** via event queues; rt_msg.c tightly couples modem input to message state

**Concepts Illustrated:**
- **Object pooling** for real-time performance
- **Priority queue** (via sorted array) for display ordering
- **Subsystem integration**: Message system acts as a **facade** between game logic and multiplayer networking
- **Frame-based vs. event-driven updates**: Timers decrement per-frame; messages added on-demand

## Potential Issues

1. **Soft-erase boundary condition**: If `UpdateMessageBackground` count wraps or if messages are added/deleted too rapidly, erased regions might not fully restore. No explicit validation that `UpdateMessageBackground >= 0` after decrement.

2. **Message overflow**: `GetFreeMessage()` assumes at least one non-permanent message exists; if all `MAXMSGS` slots are permanent, the search loop finds the message with smallest `tictime` (which is negative for permanent messages) and deletes it. This could accidentally kill important UI (e.g., player menu).

3. **Modem buffer assumption**: `SetMessage()` allocates `COM_MAXTEXTSTRINGLENGTH + 1` for permanent messages without bounds-checking that the input string fits; if a menu message exceeds this, it silently overflows.

4. **Font caching per-frame**: `DrawMessages()` calls `W_CacheLumpName("ifnt")` every frame; while the engine likely caches this, repeated cache lookups are inefficient compared to caching the pointer once at init.

5. **Race condition in multiplayer**: `FinishModemMessage()` checks `MSG.directed` and reads `numplayers` without any synchronization; if the modem state or player count changes between message composition and send, routing could fail silently.
