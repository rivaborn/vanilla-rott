# rott/cin_actr.h

## File Purpose
Public header for cinematic actor management. Declares the API for managing, rendering, and updating actors within cinematic sequences (animated sprites, backdrops, and effects displayed during game cutscenes).

## Core Responsibilities
- Maintain a linked list of active cinematic actors (via `firstcinematicactor` and `lastcinematicactor`)
- Add and remove actors from the cinematic actor list
- Allocate new cinematic actor instances
- Initialize and shut down the cinematic actor subsystem
- Spawn actors from cinematic event data
- Update and render all active cinematic actors each frame

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `actortype` | struct | Cinematic actor instance; holds effect type, effect data, and linked-list pointers |
| `enum_eventtype` | enum | Event type discriminator (sprite, backdrop, palette, flic, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `firstcinematicactor` | `actortype *` | extern global | Head of linked list of active cinematic actors |
| `lastcinematicactor` | `actortype *` | extern global | Tail of linked list of active cinematic actors |

## Key Functions / Methods

### AddCinematicActor
- Signature: `void AddCinematicActor ( actortype * actor )`
- Purpose: Insert an actor into the active cinematic actor linked list
- Inputs: Pointer to an `actortype` instance
- Outputs/Return: None
- Side effects: Modifies `firstcinematicactor` and/or `lastcinematicactor`; may relink actor in list
- Calls: Not inferable from header
- Notes: Assumes actor is not already in list; implementation likely maintains doubly-linked list via `next`/`prev` pointers

### DeleteCinematicActor
- Signature: `void DeleteCinematicActor ( actortype * actor)`
- Purpose: Remove an actor from the active cinematic actor linked list
- Inputs: Pointer to an `actortype` instance to remove
- Outputs/Return: None
- Side effects: Modifies `firstcinematicactor` and/or `lastcinematicactor`; unlinks actor from list
- Calls: Not inferable from header
- Notes: Likely does not deallocate memory; may be paired with a pool allocator

### GetNewCinematicActor
- Signature: `actortype * GetNewCinematicActor ( void )`
- Purpose: Allocate and return a new, uninitialized cinematic actor
- Inputs: None
- Outputs/Return: Pointer to a new `actortype`
- Side effects: Allocates memory; may update internal actor pool state
- Calls: Not inferable from header
- Notes: Actor is not automatically added to the active list; caller must call `AddCinematicActor`

### StartupCinematicActors
- Signature: `void StartupCinematicActors ( void )`
- Purpose: Initialize the cinematic actor subsystem (e.g., allocate pool, reset linked list)
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes global state
- Calls: Not inferable from header
- Notes: Called once at engine startup

### ShutdownCinematicActors
- Signature: `void ShutdownCinematicActors ( void )`
- Purpose: Clean up the cinematic actor subsystem and free resources
- Inputs: None
- Outputs/Return: None
- Side effects: Frees memory; resets global actor pointers
- Calls: Not inferable from header
- Notes: Called once at engine shutdown

### SpawnCinematicActor
- Signature: `void SpawnCinematicActor ( enum_eventtype type, void * effect )`
- Purpose: Create and activate a new cinematic actor from cinematic event data
- Inputs: Event type discriminator; opaque pointer to effect data (interpreted according to type)
- Outputs/Return: None
- Side effects: Allocates actor, initializes it, and adds to active list
- Calls: Likely calls `GetNewCinematicActor()` and `AddCinematicActor()` internally
- Notes: Effect pointer is cast and interpreted based on type (e.g., `spriteevent`, `backevent`, etc.)

### DrawCinematicActors
- Signature: `void DrawCinematicActors ( void )`
- Purpose: Render all active cinematic actors to the display
- Inputs: None
- Outputs/Return: None
- Side effects: Writes to video memory/framebuffer
- Calls: Not inferable from header
- Notes: Likely iterates `firstcinematicactor` → `lastcinematicactor`; called once per frame during cinematic playback

### UpdateCinematicActors
- Signature: `void UpdateCinematicActors ( void )`
- Purpose: Advance animation, scrolling, and other state for all active cinematic actors
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies actor state (frame, offset, scale, etc.); may remove completed actors
- Calls: Not inferable from header
- Notes: Likely iterates active list and updates duration, position, and animation frame counters; may trigger actor removal when complete

## Control Flow Notes
This module integrates into the cinematic playback pipeline:
1. **Initialization**: `StartupCinematicActors()` called during engine/cinematic startup
2. **Per-frame loop**: `UpdateCinematicActors()` advances state, then `DrawCinematicActors()` renders
3. **Event spawning**: `SpawnCinematicActor()` called when cinematic events are triggered (e.g., from a timeline or event queue)
4. **Actor lifecycle**: Actors are added via `AddCinematicActor()`, updated/rendered per frame, and removed via `DeleteCinematicActor()` when complete
5. **Shutdown**: `ShutdownCinematicActors()` frees resources at end of cinematic

## External Dependencies
- **Includes**: `cin_glob.h` (cinematic timing macros), `cin_def.h` (type definitions)
- **Types used**: `actortype`, `enum_eventtype` (defined in cin_def.h)
- **Macros used**: Not visible in header
- **Defined elsewhere**: `actortype` structure, `enum_eventtype` enumeration, underlying memory allocation and rendering functions
