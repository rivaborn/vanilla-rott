# rott/cin_evnt.h

## File Purpose
Public interface for cinematic event management in ROTT. Provides functions to create, manage, and process time-synchronized events (visual effects, sprite animations, backdrop scrolling, palette changes, etc.) during cinematic sequences.

## Core Responsibilities
- Event lifecycle management (create, add, delete, retrieve)
- Maintenance of a doubly-linked event queue (`firstevent`, `lastevent`)
- Time-based event scheduling and execution
- Cinematic initialization and shutdown
- Resource precaching for cinematics

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `eventtype` | struct | Linked-list node representing a single cinematic event with timing, effect type, and effect data |
| `enum_eventtype` | enum | Event type identifier (backgrounds, sprites, backdrops, palette, flic, fadeout, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `firstevent` | `eventtype *` | extern (global) | Pointer to the first event in the cinematic event queue |
| `lastevent` | `eventtype *` | extern (global) | Pointer to the last event in the cinematic event queue |

## Key Functions / Methods

### AddEvent
- Signature: `void AddEvent(eventtype * event)`
- Purpose: Insert an event into the doubly-linked event queue
- Inputs: Pointer to a pre-constructed event node
- Outputs/Return: None
- Side effects: Modifies global `firstevent`/`lastevent` pointers; maintains queue structure
- Calls: Not inferable from this file
- Notes: Event must be pre-allocated; function likely maintains time-sorted or insertion-order queue

### DeleteEvent
- Signature: `void DeleteEvent(eventtype * event)`
- Purpose: Remove an event from the doubly-linked event queue
- Inputs: Pointer to event node to remove
- Outputs/Return: None
- Side effects: Modifies global `firstevent`/`lastevent` pointers and adjacent node links
- Calls: Not inferable from this file
- Notes: Assumes valid event pointer; unlinks from queue and updates links

### GetNewEvent
- Signature: `eventtype * GetNewEvent(void)`
- Purpose: Allocate and return a new, uninitialized event structure
- Inputs: None
- Outputs/Return: Pointer to newly allocated event
- Side effects: Memory allocation
- Calls: Not inferable from this file
- Notes: Caller is responsible for initializing and adding to queue

### CreateEvent
- Signature: `eventtype * CreateEvent(int time, int type)`
- Purpose: Allocate a new event and initialize it with time and effect type
- Inputs: `time` (cinematic time offset), `type` (effect type enum)
- Outputs/Return: Pointer to initialized event
- Side effects: Memory allocation
- Calls: Likely calls `GetNewEvent` internally
- Notes: Convenience wrapper; effect data pointer likely remains uninitialized

### StartupEvents / ShutdownEvents
- Signature: `void StartupEvents(void)` / `void ShutdownEvents(void)`
- Purpose: Initialize and tear down cinematic event system
- Inputs: None
- Outputs/Return: None
- Side effects: Initialize/free global event queue and related state
- Calls: Not inferable from this file
- Notes: Called at cinematic system startup and shutdown

### ParseEvent
- Signature: `void ParseEvent(int time)`
- Purpose: Not inferable from signature alone; likely processes script/data to create events for a given time
- Inputs: `time` (cinematic time)
- Outputs/Return: None
- Side effects: Likely creates or modifies events in the queue
- Calls: Not inferable from this file

### UpdateCinematicEvents
- Signature: `void UpdateCinematicEvents(int time)`
- Purpose: Update all active events at a given cinematic time; render/apply their effects
- Inputs: `time` (current cinematic playback time)
- Outputs/Return: None
- Side effects: Rendering, palette changes, sprite updates, screen state modifications
- Calls: Not inferable from this file
- Notes: Likely iterates the event queue and executes events whose time has come

### PrecacheCinematic
- Signature: `void PrecacheCinematic(void)`
- Purpose: Load and cache cinematic assets (sprites, backdrops, palettes) into memory ahead of playback
- Inputs: None
- Outputs/Return: None
- Side effects: Memory allocation; I/O (asset loading)
- Calls: Not inferable from this file

## Control Flow Notes
Part of the cinematic/cutscene engine. Events are time-synchronized and queued during cinematic playback. Expected flow: `StartupEvents()` → `CreateEvent()` / `AddEvent()` (script load phase) → `UpdateCinematicEvents()` (per frame during playback) → `ShutdownEvents()` (cleanup). The `ParseEvent()` function likely bridges cinematic script parsing to event queue construction.

## External Dependencies
- **cin_glob.h**: Global cinematic declarations (`CinematicDelay`, `GetCinematicTime`, `CinematicAbort`)
- **cin_def.h**: Type definitions (`eventtype`, `enum_eventtype`, effect structures: `flicevent`, `spriteevent`, `backevent`, `paletteevent`)
- Indirect: `rt_def.h`, `rt_util.h`, `isr.h`, `<time.h>` (via cin_glob.h)
