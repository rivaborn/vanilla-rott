# rott/cin_evnt.c — Enhanced Analysis

## Architectural Role
cin_evnt.c acts as the **event dispatcher and scheduling layer** for the cinematic subsystem. It bridges script parsing (via scriplib) with effect/actor instantiation (cin_efct, cin_actr), maintaining a time-ordered event queue and triggering spawning at runtime. This file encapsulates the state machine that drives cinematic playback: parse-time event registration → runtime dispatch → resource cleanup.

## Key Cross-References

### Incoming (who depends on this file)
- **cin_main.c**: Calls `CreateEvent` / `ParseEvent` during script loading; calls `UpdateCinematicEvents` during playback loop
- **rt_film.c**: Calls `AddEvents` (wrapper, inferred from cross-ref index) to integrate cinematic playback into game flow
- Public API exported via **cin_evnt.h**: `AddEvent`, `CreateEvent`, `StartupEvents`, `ShutdownEvents`, `UpdateCinematicEvents`, `GetEventType`

### Outgoing (what this file depends on)
- **cin_efct.c**: Spawning functions (`SpawnCinematicBack`, `SpawnCinematicSprite`, etc.); `PrecacheCinematicEffect` for resource preloading
- **cin_actr.c**: `SpawnCinematicActor` called from `UpdateCinematicEvents` when events trigger
- **scriplib.h**: `GetToken`, `ParseNum` for script parsing during event creation
- **w_wad.h**: `W_CacheLumpName` to load graphics (lpic_t, patch_t) during parsing
- **z_zone.h**: Memory allocation via `SafeMalloc`/`SafeFree`
- **cin_glob.h, cin_def.h**: Type definitions and cinematic subsystem globals

## Design Patterns & Rationale

**Event Stream Model**: Events form a doubly-linked list ordered by time. This is not a priority queue—it's a sorted list exploited for O(n) iteration with early break in `UpdateCinematicEvents`. Chosen likely for simplicity on 1990s hardware and predictable insertion order (scripts define events in time order).

**Eager Effect Instantiation**: Parsing immediately spawns effect objects (via `SpawnCinematic*`), rather than deferring to playback. This simplifies the update loop (only actors spawn at runtime) but increases initialization latency. The `PrecacheCinematic` function mitigates this by loading resources before playback starts.

**Doubly-Linked List over Array**: Enables O(1) deletion during iteration (critical in `UpdateCinematicEvents`), and allows bidirectional traversal if needed. Given bounded event count (MAXCINEMATICEVENTS), this is over-engineered vs. a simple array, suggesting defensive design or legacy reuse.

**Type-Specific Parsing**: `GetEventType` → `ParseEvent` → type-specific `ParseBack`/`ParseSprite`/etc. dispatch. This separates concerns but creates multiple global token-parsing side effects.

## Data Flow Through This File

1. **Parse Phase** (offline):
   - Script tokens → `GetEventType` → `ParseEvent` → type-specific parser
   - Parser reads parameters, caches graphics (`W_CacheLumpName`), spawns effect objects
   - Newly created events linked into global list by time

2. **Precache Phase** (just before playback):
   - `PrecacheCinematic` traverses event list
   - Calls `PrecacheCinematicEffect` to lock resources in memory

3. **Update Phase** (per frame):
   - `UpdateCinematicEvents(currentTime)` iterates list from head
   - When `event->time == currentTime`: spawn actor via `SpawnCinematicActor`, delete event node
   - Stops early if `event->time > currentTime` (list is time-sorted)

4. **Shutdown**:
   - `ShutdownEvents` iterates remaining events, deallocates all

## Learning Notes

**Idiomatic to this era**: 
- Parsing directly triggers resource allocation (no deferred instantiation pattern)
- Callback-style update (game loop calls `UpdateCinematicEvents` each frame) rather than event-driven
- Doubly-linked list is overkill for a small, ordered collection—suggests either reuse from a more general list library or defensive over-engineering

**Modern alternatives**:
- Modern engines use a Timeline/Tween system (Unreal's Sequencer, Godot's AnimationPlayer) that separates playback logic from per-frame update
- Would use a heap/priority queue for true arbitrary event ordering, or a flat array with binary search
- Lazy instantiation (spawn effects only when their time arrives) is standard

**Connections to game engine concepts**:
- This is a **scripted timeline system** (declarative sequence of time-stamped actions)
- Related to animation curves, cinematics, and event scheduling—all foundational to any engine with cutscenes or demo playback
- The doubly-linked list pattern also appears in actor management (e.g., rt_actor.c likely uses similar doubly-linked active/inactive lists)

## Potential Issues

- **Buffer Overflow Risk**: `ParseBack`, `ParseSprite`, `ParseFlic`, `ParsePalette` all use fixed 10-byte `char name[10]` with unbounded `strcpy(name, token)`. A script token >9 chars causes stack corruption.
- **No Null Checks on List Operations**: `DeleteEvent` and `AddEvent` assume non-NULL pointers; a corrupted event pointer could crash during update.
- **Implicit Time Ordering**: Events must be parsed in time order (earliest first) for the early-break optimization in `UpdateCinematicEvents` to work. No runtime validation of this assumption.
- **Resource Leak on Parse Failure**: If `ParseEvent` errors mid-parse (e.g., missing token), the partially-constructed event may remain in the list with a NULL effect pointer.
