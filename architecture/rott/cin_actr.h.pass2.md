# rott/cin_actr.h — Enhanced Analysis

## Architectural Role

This header is the public API for the **cinematic actor subsystem**, which manages animated sprites and visual effects rendered during gameplay cutscenes. It sits at the boundary between the **cinematic event system** (`cin_evnt.*`) and the **rendering pipeline**, providing lifecycle and frame-by-frame update services for all active cinematic visual elements. The dual linked-list structure (first/last pointers) suggests actors persist across frames and are processed in order.

## Key Cross-References

### Incoming (who depends on this file)
- **`cin_main.c`**: Likely calls `StartupCinematicActors()` / `ShutdownCinematicActors()` during cinematic session lifecycle
- **`cin_main.c`** or **cinematic loop**: Calls `UpdateCinematicActors()` and `DrawCinematicActors()` per frame during playback
- **`cin_evnt.c`**: Event dispatcher calls `SpawnCinematicActor()` when cinematic events are triggered (sprites, backdrops, palette effects, FLICs)
- **Possible: `rt_film.c`** (`AddEvents` function exists there, suggesting film/movie playback uses cinematic actors)

### Outgoing (what this file depends on)
- **`cin_def.h`**: Defines `actortype` struct and `enum_eventtype` discriminator
- **`cin_glob.h`**: Provides cinematic timing macros and global state (abort flags, delay counters)
- **Implicit**: Memory allocator (likely a pool allocator in `.c` implementation)
- **Implicit**: Video/rendering backend (called from `DrawCinematicActors()`)

## Design Patterns & Rationale

| Pattern | Evidence | Rationale |
|---------|----------|-----------|
| **Linked-list object pool** | `first/lastcinematicactor` pointers; `Add`/`Delete` operations | O(1) insertion/removal; suitable for variable-count, short-lived visual effects |
| **Type-driven spawning** | `SpawnCinematicActor(enum_eventtype, void*)` | Opaque effect pointer cast based on type avoids inheritance overhead in C; common in 1990s game code |
| **Separation of allocation & activation** | `GetNewCinematicActor()` returns unlinked actor; caller must `Add` it | Decouples memory management from active-list management; enables reuse patterns |
| **Update-render decoupling** | Separate `Update` and `Draw` calls | Standard game loop pattern; allows partial updates (e.g., draw without updating) |

**Why this structure?** This is a 1990s game engine with hard real-time constraints. Object pooling avoids allocation overhead mid-frame. Type-driven dispatching avoids virtual function overhead. Linked lists allow arbitrary actor counts without pre-allocation.

## Data Flow Through This File

```
Cinematic Event Stream (cin_evnt.c)
    ↓
    | SpawnCinematicActor(type, effect_data)
    ↓
GetNewCinematicActor() [allocate from pool]
    ↓
AddCinematicActor() [link into active list]
    ↓
Per-frame Loop:
  1. UpdateCinematicActors() [advance animation frame, duration, etc.]
  2. DrawCinematicActors() [render linked list to framebuffer]
    ↓
DeleteCinematicActor() [unlink when duration expires or event completes]
```

State lives entirely in the linked list; no external registry consulted.

## Learning Notes

**Idiomatic to 1990s engines:**
- No dynamic dispatch; everything routed through `enum_eventtype` switch
- Global head/tail pointers instead of a manager object (C idiom; avoids indirection)
- Caller responsible for linking/unlinking (manual vs. RAII)
- Void pointer for effect data (classic C polymorphism)

**Modern counterpart:** Would use a scene graph node or Entity Component System (ECS) with cinematic tags; effects would have a component with lifetime and rendering info.

**Key learning:** Cinematic actors are **separate from gameplay actors** (which likely live in `rt_actor.h`). This reflects a common design: cutscenes run in isolation with their own object pool to avoid interference with game state. Note related function `AddAnimStatic` in `rt_stat.h` — static sprites in the game world use a different system.

## Potential Issues

1. **No reference counting or validity checking visible**: If `DeleteCinematicActor()` is called on a dangling pointer, silent corruption is possible. First-pass noted this "likely does not deallocate memory," suggesting a pool. Pool must guard against double-deletion.

2. **Type safety on void pointer**: `SpawnCinematicActor(enum_eventtype, void*)` — if event type and effect data misalign at call site, undefined behavior ensues. No verification in header.

3. **Unguarded global pointers**: Extern globals `first/lastcinematicactor` can be modified by any code in the cinematic subsystem. Concurrent access or accidental modification is possible if not disciplined.
