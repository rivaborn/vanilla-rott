# rott/cin_evnt.c

## File Purpose
Manages the lifecycle of cinematic events: creating, storing, parsing, and processing time-based events for cutscenes. Events are stored in a doubly-linked list and triggered by the update loop when the current playback time matches their scheduled time.

## Core Responsibilities
- Maintain a doubly-linked list of cinematic events (add, delete, create)
- Parse cinematic event definitions from script tokens
- Trigger events at their scheduled times during playback
- Pre-cache resources for all events in a cinematic
- Manage system startup and shutdown

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| eventtype | struct | Doubly-linked list node holding event metadata and effect pointer |
| enum_eventtype | enum | Event type classifier (backgrounds, sprites, palette, flic, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| firstevent | eventtype* | global | Head pointer of event linked list |
| lastevent | eventtype* | global | Tail pointer of event linked list |
| numevents | int | static | Counter of allocated events; checked against MAXCINEMATICEVENTS |
| eventsystemstarted | boolean | static | Prevents double-initialization |

## Key Functions / Methods

### AddEvent
- **Signature:** `void AddEvent (eventtype * event)`
- **Purpose:** Append event to the end of the doubly-linked list.
- **Inputs:** Pointer to initialized eventtype.
- **Outputs/Return:** None (modifies global state).
- **Side effects:** Updates `firstevent` and `lastevent` globals.
- **Calls:** None (direct pointer assignments).
- **Notes:** Assumes caller provides valid, initialized event; no NULL checks.

### DeleteEvent
- **Signature:** `void DeleteEvent(eventtype * event)`
- **Purpose:** Remove event from linked list and deallocate its memory.
- **Inputs:** Pointer to eventtype in the list.
- **Outputs/Return:** None.
- **Side effects:** Unlinks from list; calls `SafeFree`; updates `firstevent`/`lastevent` if needed.
- **Calls:** `SafeFree` (z_zone.h).
- **Notes:** Handles boundary cases (first/last node); sets next/prev to NULL before freeing.

### GetNewEvent
- **Signature:** `eventtype * GetNewEvent ( void )`
- **Purpose:** Allocate and initialize a new event node.
- **Inputs:** None.
- **Outputs/Return:** Pointer to newly allocated eventtype.
- **Side effects:** Increments `numevents`; calls `SafeMalloc`; may call `Error` if count exceeds MAXCINEMATICEVENTS.
- **Calls:** `SafeMalloc` (z_zone.h), `Error` (defined elsewhere).
- **Notes:** Initializes next/prev pointers to NULL.

### StartupEvents / ShutdownEvents
- **Signatures:** `void StartupEvents ( void )` / `void ShutdownEvents ( void )`
- **Purpose:** Initialize and clean up the event subsystem.
- **Side effects:** `StartupEvents` sets flags and clears list; `ShutdownEvents` iterates list calling `DeleteEvent`.
- **Calls:** `DeleteEvent` (from ShutdownEvents).
- **Notes:** Both are idempotent; check `eventsystemstarted` flag on entry.

### CreateEvent
- **Signature:** `eventtype * CreateEvent ( int time, int type )`
- **Purpose:** Factory function: allocate, initialize, and register a new event.
- **Inputs:** `time` (playback time), `type` (enum_eventtype).
- **Outputs/Return:** Pointer to created event.
- **Side effects:** Calls `GetNewEvent` and `AddEvent`; modifies linked list.
- **Calls:** `GetNewEvent`, `AddEvent`.
- **Notes:** Sets `effect` pointer to NULL; caller must populate via parser.

### GetEventType
- **Signature:** `enum_eventtype GetEventType ( void )`
- **Purpose:** Map script token strings to enum_eventtype values.
- **Inputs:** Uses global `token` variable (from scriplib).
- **Outputs/Return:** Corresponding enum_eventtype (or -1 on error).
- **Side effects:** Calls `GetToken` to fetch script tokens; may call `Error`.
- **Calls:** `GetToken` (scriplib.h), `strcmpi`, `Error`.
- **Notes:** Handles multi-token sequences (e.g., "BACKGROUND SCROLL"); returns -1 on unrecognized token.

### ParseBack / ParseSprite / ParseFlic / ParsePalette
- **Signature:** `void ParseBack/ParseSprite/ParseFlic/ParsePalette ( eventtype * event )`
- **Purpose:** Extract type-specific parameters from script and create corresponding effect object.
- **Side effects:** Call `GetToken` and `ParseNum`; allocate effects via `SpawnCinematic*` functions; **buffer overflow risk in strcpy (10-byte fixed buffers)**.
- **Calls:** `GetToken`, `strcpy`, `ParseNum` (scriplib), `W_CacheLumpName` (w_wad), `SpawnCinematic*` (cin_efct).
- **Notes:** ParseBack handles multiple branches (scrolling vs. non-scrolling backgrounds, multi-layer); ParseFlic parses LOOP/NOLOOP and FILE/LUMP flags.

### ParseEvent
- **Signature:** `void ParseEvent ( int time )`
- **Purpose:** Dispatcher: create event and route to appropriate type-specific parser.
- **Inputs:** `time` (scheduled event time).
- **Outputs/Return:** None.
- **Side effects:** Creates event via `CreateEvent`, calls type-specific parser.
- **Calls:** `CreateEvent`, `GetEventType`, `ParseBack`, `ParseSprite`, `ParseFlic`, `ParsePalette` (conditionally).
- **Notes:** Switch statement over `event->effecttype`; some cases (fadeout, cinematicend, etc.) require no parsing.

### UpdateCinematicEvents
- **Signature:** `void UpdateCinematicEvents ( int time )`
- **Purpose:** Find and trigger all events scheduled for the given time.
- **Inputs:** Current playback `time`.
- **Outputs/Return:** None.
- **Side effects:** Iterates linked list; deletes triggered events; calls `SpawnCinematicActor`.
- **Calls:** `SpawnCinematicActor` (cin_actr), `DeleteEvent`.
- **Notes:** Early break when `event->time > time` (implies list is time-sorted); safe removal during iteration.

### PrecacheCinematic
- **Signature:** `void PrecacheCinematic ( void )`
- **Purpose:** Pre-load all graphics/resources for the entire cinematic before playback.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Iterates all events; calls resource loader for each.
- **Calls:** `PrecacheCinematicEffect` (cin_efct).
- **Notes:** Called before cinematic playback to avoid stalls during runtime.

## Control Flow Notes
**Initialization phase:** `StartupEvents()` initializes globals; `ParseEvent()` populates list during script parsing.  
**Update loop:** `UpdateCinematicEvents(time)` fires events at scheduled times, spawning actors and deleting processed nodes.  
**Shutdown:** `ShutdownEvents()` deallocates all remaining events.  
Events appear to be inserted in time order, enabling O(n) traversal with early break in update.

## External Dependencies
- **cin_glob.h, cin_efct.h, cin_actr.h, cin_def.h:** Cinematic system definitions and effect/actor spawning.
- **scriplib.h:** Script parsing (`GetToken`, `ParseNum`).
- **w_wad.h:** WAD resource caching (`W_CacheLumpName`); graphics types (lpic_t, patch_t).
- **z_zone.h:** Memory management (`SafeMalloc`, `SafeFree`).
- **string.h:** C standard string functions (`strcpy`, `strcmpi`).
- **memcheck.h:** Memory debugging.

**Security note:** Fixed-size buffers (10 bytes) in parse functions with unbounded `strcpy` calls create potential buffer overflows.
