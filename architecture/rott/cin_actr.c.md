# rott/cin_actr.c

## File Purpose
Manages a doubly-linked list of cinematic actors that encapsulate visual effects during cinematic sequences. Each actor holds a type and opaque effect data pointer. Provides lifecycle management (creation, deletion, update, rendering) for actors within the cinematic system.

## Core Responsibilities
- Maintain a global linked list of active cinematic actors (head/tail pointers)
- Allocate and deallocate actor objects with bounds checking
- Insert actors into the linked list and remove them with proper pointer bookkeeping
- Initialize and shutdown the cinematic actor system (state reset)
- Spawn new actors with specific effect types and data payloads
- Update all actors each frame, removing those marked as complete
- Render all actors in layered phases (screen functions → background → sprites → backdrop → foreground → palette)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `actortype` | struct | Represents a cinematic actor; holds effect type, opaque effect data pointer, and linked-list pointers |
| `enum_eventtype` | enum | (from cin_def.h) Defines 13 cinematic effect types: backgrounds, sprites, backdrops, palette, flic, fade, etc. |
| `enum_drawphases` | enum | (local to DrawCinematicActors) Defines 7 rendering phases for layered composition |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `firstcinematicactor` | `actortype*` | global | Head pointer of linked list of active cinematic actors |
| `lastcinematicactor` | `actortype*` | global | Tail pointer of linked list for O(1) append |
| `cinematicactorsystemstarted` | boolean | static | Tracks whether the actor system has been initialized; gates startup/shutdown |
| `numcinematicactors` | int | static | Counter of allocated actors; used for overflow detection (max 30) |

## Key Functions / Methods

### AddCinematicActor
- **Signature:** `void AddCinematicActor(actortype * actor)`
- **Purpose:** Append actor to the tail of the linked list
- **Inputs:** `actor` – actor object to add
- **Outputs/Return:** None
- **Side effects:** Modifies `lastcinematicactor` and `firstcinematicactor` pointers; updates actor's `prev` pointer if list is non-empty
- **Calls:** (none visible)
- **Notes:** O(1) append due to tail pointer; handles empty list (sets both first and last)

### DeleteCinematicActor
- **Signature:** `void DeleteCinematicActor(actortype * actor)`
- **Purpose:** Remove actor from linked list, free effect data and actor object
- **Inputs:** `actor` – actor to remove
- **Outputs/Return:** None
- **Side effects:** Modifies list pointers; deallocates `actor->effect` (if non-NULL) and `actor`; updates first/last pointers if needed
- **Calls:** `SafeFree()`
- **Notes:** Handles removal of head, tail, and middle nodes; sets actor's pointers to NULL (not strictly necessary before freeing)

### GetNewCinematicActor
- **Signature:** `actortype * GetNewCinematicActor(void)`
- **Purpose:** Allocate a new cinematic actor, increment counter, add to list
- **Inputs:** None
- **Outputs/Return:** Pointer to newly allocated and initialized actor
- **Side effects:** Increments `numcinematicactors`; calls `SafeMalloc()`; calls `AddCinematicActor()`
- **Calls:** `SafeMalloc()`, `AddCinematicActor()`
- **Notes:** Calls `Error()` if actor count exceeds MAXCINEMATICACTORS (30); initializes next/prev pointers to NULL

### StartupCinematicActors
- **Signature:** `void StartupCinematicActors(void)`
- **Purpose:** Initialize the cinematic actor system
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `cinematicactorsystemstarted` to true; initializes list pointers and actor counter to null/zero; idempotent (early return if already started)
- **Calls:** (none)
- **Notes:** Guard flag prevents redundant reinitializations

### ShutdownCinematicActors
- **Signature:** `void ShutdownCinematicActors(void)`
- **Purpose:** Clean up and destroy all active cinematic actors
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `cinematicactorsystemstarted` to false; iterates list and calls `DeleteCinematicActor()` on each actor; idempotent
- **Calls:** `DeleteCinematicActor()`
- **Notes:** Safe iteration using saved `nextactor` pointer before deletion; early return if not started

### SpawnCinematicActor
- **Signature:** `void SpawnCinematicActor(enum_eventtype type, void * effect)`
- **Purpose:** Create a new actor with given effect type and data
- **Inputs:** `type` – effect type enum; `effect` – opaque pointer to effect-specific data struct
- **Outputs/Return:** None (effect data passed in; caller retains ownership pattern unclear)
- **Side effects:** Allocates new actor via `GetNewCinematicActor()`; stores type and effect pointer
- **Calls:** `GetNewCinematicActor()`
- **Notes:** Simple factory; does not validate type or effect pointer

### UpdateCinematicActors
- **Signature:** `void UpdateCinematicActors(void)`
- **Purpose:** Update all cinematic actors; remove those that have completed
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Delegates to `UpdateCinematicEffect()` for each actor; deletes actors returning false (completed); modifies list
- **Calls:** `UpdateCinematicEffect()`, `DeleteCinematicActor()`
- **Notes:** Safe iteration with saved `nextactor` pointer; handles deletion during iteration

### DrawCinematicActors
- **Signature:** `void DrawCinematicActors(void)`
- **Purpose:** Render all cinematic actors in layered phases for proper composition
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `DrawCinematicEffect()` for selected actors; calls `XFlipPage()` at end of phase iteration; may delete actors if drawing returns false; conditional debug output (DUMP flag)
- **Calls:** `DrawCinematicEffect()`, `DeleteCinematicActor()`, `XFlipPage()`
- **Notes:** 
  - Outer loop iterates 7 phases (screenfunctions → palettefunctions)
  - Inner loop iterates actors; switch on actor type determines which phase draws it
  - Screen functions (fadeout, blankscreen, etc.) and palette ops prevent page flip (`flippage=false`)
  - Background and sprite phases allow page flip for efficient rendering
  - Safe deletion during iteration using saved `nextactor`

## Control Flow Notes
This module is part of the cinematic rendering pipeline, typically called during a cinematic sequence or cutscene:

1. **Initialization:** `StartupCinematicActors()` called at cinematic start
2. **Per-frame loop:**
   - `UpdateCinematicActors()` – update state, mark completed actors
   - `DrawCinematicActors()` – render in phases, delete completed actors
3. **Shutdown:** `ShutdownCinematicActors()` called at end of cinematic
4. Actors do not update/draw themselves; they delegate to external effect handlers (defined in cin_efct.c), providing a lightweight container pattern

## External Dependencies
- **Headers:** `cin_glob.h`, `cin_def.h`, `cin_actr.h`, `cin_efct.h`, `modexlib.h`, `memcheck.h`
- **Defined elsewhere:**
  - `SafeMalloc()`, `SafeFree()` – memory management from memcheck.h
  - `UpdateCinematicEffect()`, `DrawCinematicEffect()` – effect handlers from cin_efct.c
  - `XFlipPage()` – video/display update from modexlib.c
  - `Error()` – error reporting (stdio or engine)
  - `MAXCINEMATICACTORS` constant (30, from cin_def.h)
  - `enum_eventtype`, `actortype` – from cin_def.h
