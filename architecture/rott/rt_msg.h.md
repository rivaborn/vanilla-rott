# rott/rt_msg.h

## File Purpose
Header for the in-game message/dialog system in Rise of the Triad. Defines message types, priority flags, and the message queue infrastructure for displaying game events, system messages, modem communication, and cheat notifications.

## Core Responsibilities
- Define message type constants and flag bits (priority, permanence, deletion policy)
- Declare the global message queue and enable/disable toggle
- Provide message lifecycle functions (add, delete, render, clear)
- Support modem/network message input and editing
- Handle message background restoration and priority-based deletion

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `messagetype` | struct | Container for a single message: active flag, flags (priority/permanence), display time (tics), and text pointer |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Messages` | `messagetype[MAXMSGS]` | global extern | Fixed-size message queue (max 15 messages) |
| `MessagesEnabled` | `boolean` | global extern | Toggle to enable/disable message rendering |

## Key Functions / Methods

### AddMessage
- Signature: `int AddMessage(char *text, int flags)`
- Purpose: Enqueue a new message with specified type/flags
- Inputs: Message text (C string), flags (message type + priority bits)
- Outputs/Return: Message index or error code (inferred)
- Side effects: Modifies `Messages` array; may trigger priority-based deletion
- Calls: (not inferable from header)
- Notes: Flags encode both message type and attributes (permanence, node-delete policy, priority)

### DrawMessages
- Signature: `void DrawMessages(void)`
- Purpose: Render all active messages to screen
- Inputs: None (reads from global `Messages`)
- Outputs/Return: None
- Side effects: Modifies framebuffer/display
- Calls: (not inferable from header)
- Notes: Called once per frame; respects `MessagesEnabled` flag

### DeleteMessage
- Signature: `void DeleteMessage(int num)`
- Purpose: Deactivate and remove a message by index
- Inputs: Message index
- Outputs/Return: None
- Side effects: Clears `Messages[num]`
- Calls: (not inferable from header)
- Notes: Respects `MSG_NODELETE` flag to prevent forced deletion

### RestoreMessageBackground
- Signature: `void RestoreMessageBackground(void)`
- Purpose: Redraw the game screen area behind messages (screen refresh helper)
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies framebuffer
- Calls: (not inferable from header)

### InitializeMessages
- Signature: `void InitializeMessages(void)`
- Purpose: Clear/reset message queue at startup or level transition
- Inputs: None
- Outputs/Return: None
- Side effects: Clears global `Messages`

### UpdateModemMessage / ModemMessageDeleteChar / FinishModemMessage
- Purpose: Support live text entry for modem messages; editing helpers
- Notes: Used during multiplayer setup or chat input; operate on a single message being composed

### ResetMessageTime
- Purpose: Refresh display timer for messages (extend visible duration)

### StringLength, DeletePriorityMessage
- Purpose: Utility for message layout and automatic cleanup when queue is full

## Control Flow Notes
Integrated into the main game loop: `DrawMessages()` called during render phase; `AddMessage()` called by game logic (pickups, damage, system events, modem input); message timer decrements each frame. Permanent messages (e.g., `MSG_MODEM`, `MSG_NAMEMENU`) persist until explicitly deleted. Deletion respects priority bits and `MSG_NODELETE` flag.

## External Dependencies
- `byte`, `boolean`, `int`, `char *` — standard C types (defined elsewhere, likely `rt_types.h`)
- No explicit external module dependencies in this header; message text likely allocated by caller
