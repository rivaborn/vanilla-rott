# rott/rt_msg.c

## File Purpose
Implements the on-screen message and notification system for Rise of the Triad. Manages creation, display timing, rendering, and lifecycle of temporary notifications (pickups, cheats, system alerts) and permanent UI messages (multiplayer chat, player selection menus). Handles color-coding by message type and background restoration for erased message regions.

## Core Responsibilities
- Initialize and reset the message system
- Manage message queue with fixed-size array and slot reuse
- Add, delete, and update messages with type-based priority
- Sort and order messages for display
- Render messages with color-coding based on message flags (type)
- Update message timers and auto-expire non-permanent messages each frame
- Handle modem/multiplayer chat input and character editing
- Render player selection menus for directed messaging
- Restore screen background after messages are erased

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `messagetype` | struct | Single message with active state, text, display time, and flags (defined in _rt_msg.h) |
| `pic_t` | struct | Picture/sprite graphics data (from lumpy.h) |
| `cfont_t` | struct | Colored font with palette and character metrics (from lumpy.h) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Messages[MAXMSGS]` | `messagetype[]` | global | Array of all message slots (active and inactive) |
| `MessageSystemStarted` | int (static) | static | Flag indicating if system initialized; prevents duplicate startup |
| `LastMessageTime` | int (static) | static | Timestamp of last `UpdateMessages()` call; used to compute frame delta |
| `UpdateMessageBackground` | int (static) | static | Counter tracking how many frames to restore background regions |
| `EraseMessage[MAXMSGS]` | byte[] (static) | static | Erase counter for each message position; decrements each restore frame |
| `MessageOrder[MAXMSGS]` | int[] (static) | static | Indices into `Messages[]` sorted by display time (tictime); valid range [0..TotalMessages) |
| `TotalMessages` | int (static) | static | Count of currently active messages in `MessageOrder[]` |
| `MsgPos` | int (static) | static | Position offset for menu messages; incremented to stagger vertical placement |
| `MessagesEnabled` | boolean | global | Toggle to suppress non-essential messages during certain game states |

## Key Functions / Methods

### InitializeMessages
- **Signature:** `void InitializeMessages(void)`
- **Purpose:** Initialize or reset the message system to clean state. Called once on engine startup and before each game/level.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears all message slots, resets timers and counters, prints debug message on first init.
- **Calls:** `SafeFree()`, `memset()`
- **Notes:** Tracks first initialization with `MessageSystemStarted` to print startup message only once. Deactivates all messages and resets `TotalMessages` to 0.

### AddMessage
- **Signature:** `int AddMessage(char *text, int flags)`
- **Purpose:** Create a new message and add it to the message queue.
- **Inputs:** `text` (message string), `flags` (message type/priority, e.g., `MSG_GAME`, `MSG_CHEAT`)
- **Outputs/Return:** Index of allocated message slot
- **Side effects:** Allocates memory, may delete lower-priority messages if no free slots.
- **Calls:** `GetFreeMessage()`, `SetMessage()`, `DeletePriorityMessage()`
- **Notes:** Unless `MSG_NODELETE` flag is set, deletes existing messages with same flags to avoid duplicates. Asserts `MessageSystemStarted` is true.

### GetFreeMessage
- **Signature:** `int GetFreeMessage(void)`
- **Purpose:** Find an available message slot or free one by deleting the oldest non-permanent message.
- **Inputs:** None
- **Outputs/Return:** Index of free message slot
- **Side effects:** May delete a message if no free slots available.
- **Calls:** `DeleteMessage()`
- **Notes:** Prefers already-inactive slots; if none found, searches for message with smallest `tictime` and deletes it. Returns index even if deletion was necessary.

### SetMessage
- **Signature:** `void SetMessage(int num, char *text, int flags)`
- **Purpose:** Configure a message slot with text and properties.
- **Inputs:** `num` (message index), `text` (string), `flags` (message type)
- **Outputs/Return:** None
- **Side effects:** Allocates memory for text, marks message active, updates background erase counters.
- **Calls:** `StringLength()`, `SafeMalloc()`, `memset()`, `memcpy()`, `GetMessageOrder()`
- **Notes:** Permanent messages (checked via `PERMANENT_MSG(flags)` macro) use fixed buffer size and negative tictime; temporary messages get `MESSAGETIME` countdown. Re-calculates `MessageOrder[]` after update.

### DeleteMessage
- **Signature:** `void DeleteMessage(int num)`
- **Purpose:** Remove a message from the queue and free its memory.
- **Inputs:** `num` (message index)
- **Outputs/Return:** None
- **Side effects:** Frees text memory, sets message inactive, marks background region for redraw, recalculates `MessageOrder[]`.
- **Calls:** `SafeFree()`, `memset()`, `GetMessageOrder()`
- **Notes:** Also updates `UpdateMessageBackground` counter to signal background restoration needed for this message's display area.

### DeletePriorityMessage
- **Signature:** `void DeletePriorityMessage(int flags)`
- **Purpose:** Delete all active messages with a specific flag/type.
- **Inputs:** `flags` (message type to match)
- **Outputs/Return:** None
- **Side effects:** Calls `DeleteMessage()` on matching entries.
- **Calls:** `DeleteMessage()`
- **Notes:** Used to prevent duplicate messages of same type from appearing simultaneously.

### GetMessageOrder
- **Signature:** `void GetMessageOrder(void)`
- **Purpose:** Rebuild the `MessageOrder[]` array by sorting active messages by `tictime`.
- **Inputs:** None
- **Outputs/Return:** None (updates `MessageOrder[]` and `TotalMessages`)
- **Side effects:** Scans all message slots and populates `MessageOrder[]` in ascending tictime order.
- **Calls:** None (core sorting logic)
- **Notes:** Invariant: after this call, `MessageOrder[0..TotalMessages-1]` contains valid indices, and `MessageOrder[TotalMessages..]` is unchanged. Permanent messages have negative tictime to ensure they display first.

### UpdateMessages
- **Signature:** `void UpdateMessages(void)`
- **Purpose:** Per-frame update: decrement message timers and remove expired messages.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies `tictime` for all non-permanent messages; deletes those with `tictime <= 0`.
- **Calls:** `DeleteMessage()`
- **Notes:** Skipped if `GamePaused` is true. Frame delta computed from `LastMessageTime` and current `ticcount`.

### DrawMessages
- **Signature:** `void DrawMessages(void)`
- **Purpose:** Render all active messages to screen during render phase.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Caches font, calls display and update functions.
- **Calls:** `W_CacheLumpName()`, `DisplayMessage()`, `UpdateMessages()`
- **Notes:** Called once per frame. Loads color font (`"ifnt"`) and draws each message in order from `MessageOrder[]`.

### DisplayMessage
- **Signature:** `void DisplayMessage(int num, int position)`
- **Purpose:** Render a single message to screen with position, color, and prefix based on message flags.
- **Inputs:** `num` (message index), `position` (vertical position in message list, 0-based)
- **Outputs/Return:** None
- **Side effects:** Sets `PrintX`, `PrintY`, and `fontcolor` globals; draws text via `DrawIString()`.
- **Calls:** `DrawIString()`, `SHOW_TOP_STATUS_BAR()`, `DeleteMessage()`
- **Notes:** Each position is spaced 9 pixels vertically. If `MessagesEnabled` is false, only allows certain message types (system, menu, remote) to display; deletes others. Color selected by switch on `flags`: white (remote), light blue + prefix (modem), green (game/door/bonus), yellow (cheat), red (system/quit/macro).

### RestoreMessageBackground
- **Signature:** `void RestoreMessageBackground(void)`
- **Purpose:** Erase previously drawn messages from screen buffer by redrawing background tiles.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Redrawps background tiles and CPU jape (status graphic) for erased message regions. Decrements `UpdateMessageBackground` and `EraseMessage[]`.
- **Calls:** `W_CacheLumpName()`, `DrawTiledRegion()`, `DrawCPUJape()`
- **Notes:** Iterates through `EraseMessage[]` array; processes one erased region per frame to amortize cost. Handles view size checks (if `viewsize < 15`, use tiles; if `viewsize == 0`, redraw CPU jape if overlapped).

### UpdateModemMessage
- **Signature:** `void UpdateModemMessage(int num, char c)`
- **Purpose:** Append a character to a modem message being typed by player.
- **Inputs:** `num` (message index), `c` (character to append)
- **Outputs/Return:** None
- **Side effects:** Updates message text, appends cursor underscore, increments `MSG.length`, marks background for redraw.
- **Calls:** Direct array/global updates, background update counter.
- **Notes:** Used during multiplayer chat input. Cursor shown as underscore at end of text.

### ModemMessageDeleteChar
- **Signature:** `void ModemMessageDeleteChar(int num)`
- **Purpose:** Remove the last character from a modem message being typed.
- **Inputs:** `num` (message index)
- **Outputs/Return:** None
- **Side effects:** Decrements `MSG.length`, clears last char, repositions cursor underscore, marks background for redraw.
- **Calls:** Direct array/global updates.
- **Notes:** Backspace handling for chat input.

### FinishModemMessage
- **Signature:** `void FinishModemMessage(int num, boolean send)`
- **Purpose:** Finalize a modem message: remove cursor, optionally send or display player selection menu, clean up.
- **Inputs:** `num` (message index), `send` (true to send/route message, false to cancel)
- **Outputs/Return:** None
- **Side effects:** Removes cursor underscore, may show player selection menu, calls `AddRemoteRidiculeCommand()` or `AddTextMessage()` to route message, deletes message slot.
- **Calls:** `DrawPlayerSelectionMenu()`, `AddRemoteRidiculeCommand()`, `AddTextMessage()`, `DeletePriorityMessage()`, `DeleteMessage()`
- **Notes:** If `send == true` and `MSG.directed == true` and not in menu, shows player list before sending. Otherwise sends immediately or cancels.

### DrawPlayerSelectionMenu
- **Signature:** `void DrawPlayerSelectionMenu(void)`
- **Purpose:** Display a menu of players for directing a chat message.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Adds multiple permanent messages to display player list. Increments `MsgPos` for staggered positioning.
- **Calls:** `AddMessage()` (multiple calls)
- **Notes:** Creates entries for "Press 0-9 to select", each other player with number key, optionally "T - All team members" and "A - All players" depending on game state. Resets `MsgPos` to 0 at end.

### ResetMessageTime
- **Signature:** `void ResetMessageTime(void)`
- **Purpose:** Reset the last message update timestamp.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `LastMessageTime` to current `ticcount`.
- **Calls:** None (direct assignment)
- **Notes:** Called to prevent large time deltas (e.g., after pause or load).

### StringLength
- **Signature:** `int StringLength(char *string)`
- **Purpose:** Calculate the length of a null-terminated string including null terminator.
- **Inputs:** `string` (pointer to char array)
- **Outputs/Return:** Length in bytes (includes null byte)
- **Side effects:** None
- **Calls:** None
- **Notes:** Simple loop-based implementation; not stdlib `strlen()` (which excludes null terminator).

## Control Flow Notes
**Initialization Phase:** `InitializeMessages()` called once during engine startup to prepare message slots and zero out state.

**Per-Frame Update:**
1. `DrawMessages()` called during render:
   - Loads font resource
   - Iterates `MessageOrder[]` and calls `DisplayMessage()` for each active message
   - Calls `UpdateMessages()` to decrement timers and auto-expire temporary messages
2. `RestoreMessageBackground()` called to erase previous frames' messages from screen buffer

**Event-Driven (Game Logic):**
- `AddMessage()` called when game event occurs (cheat input, pickup, system alert, etc.)
- May trigger `DeletePriorityMessage()` to remove duplicates

**Multiplayer/Chat Input:**
- `UpdateModemMessage()` / `ModemMessageDeleteChar()` called per keystroke during chat typing
- `FinishModemMessage()` called to commit/send chat or show player selection menu

**Message Lifetime:**
- Temporary messages: created with `MESSAGETIME` countdown, decremented each frame, auto-deleted when `tictime <= 0`
- Permanent messages: created with negative `tictime` (not decremented), manually deleted via `DeleteMessage()` or `DeletePriorityMessage()`

## External Dependencies
- **rt_def.h** — Core engine types and constants (MAXMSGS, boolean, byte, memset, Error)
- **rt_view.h** — Display/viewport utilities (SHOW_TOP_STATUS_BAR, YOURCPUSUCKS_Y, fontcolor, egacolor[])
- **z_zone.h** — Memory allocation (SafeMalloc, SafeFree)
- **w_wad.h** — Resource caching (W_CacheLumpName, "backtile", "ifnt")
- **lumpy.h** — Graphics structures (pic_t, cfont_t)
- **rt_vid.h** — Rendering primitives (DrawIString, DrawTiledRegion, DrawCPUJape)
- **rt_com.h** — Modem/network communication (COM_MAXTEXTSTRINGLENGTH, MSG global struct)
- **rt_net.h** / **rt_playr.h** — Multiplayer support (numplayers, consoleplayer, PLAYERSTATE[], gamestate.teamplay)
- **rt_main.h** — Game state (GamePaused, ticcount, quiet)
- **rt_menu.h**, **rt_str.h** — Menu and string utilities
- **Standard C** — mem.h (memset, memcpy), stdlib.h

---

**Notes:**
- Message display is soft-erased (background restored over 3 frames) rather than cleared instantly, reducing visual flicker.
- Message ordering is re-computed after each add/delete to maintain sort invariant.
- Permanent vs. temporary distinction via macro `PERMANENT_MSG(flags)` affects memory allocation size and timer behavior.
