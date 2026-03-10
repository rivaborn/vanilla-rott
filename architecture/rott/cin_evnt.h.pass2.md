# rott/cin_evnt.h — Enhanced Analysis

## Architectural Role
`cin_evnt.h` defines the core event scheduling subsystem for ROTT's cinematic engine. It sits between the cinematic script/data layer (`rt_film.c`, `cin_main.c`) and the rendering/state layer (`cin_util.h` for palette, visual effect handlers). Events are the mechanism by which time-sequenced visual effects (sprite animations, palette shifts, FLI playback frames, backdrop scrolls) are scheduled and triggered during playback.

## Key Cross-References

### Incoming (who depends on this file)
- `rt_film.c`: Defines `AddEvents()` which populates the event queue by parsing cinematic script data
- `cin_main.c`: Calls `StartupEvents()` / `ShutdownEvents()` during cinematic initialization
- `cin_actr.c`: Likely calls event functions to synchronize cinematic actors with visual events
- Any cinematic playback loop: Calls `UpdateCinematicEvents(time)` each frame to trigger scheduled effects

### Outgoing (what this file depends on)
- `cin_glob.h`: Calls `GetCinematicTime()`, `CinematicDelay()`, `CinematicAbort()` for time management and playback control
- `cin_def.h`: Uses `eventtype` struct and `enum_eventtype` enum to define event data and types
- `fli_main.c` (indirectly): Events may trigger FLI frame transitions via `CopyNextFlicBlock()`
- `cin_util.h` (indirectly): `UpdateCinematicEvents()` likely calls palette/effect handlers in utilities

## Design Patterns & Rationale
- **Doubly-linked queue**: Classic 1990s approach for efficient insertion/deletion at arbitrary positions. Allows time-sorted or insertion-order queueing.
- **Lazy evaluation**: Events are *created and queued* during script load phase, then *executed* on-demand during playback. Decouples parsing from rendering.
- **Separation from actors**: Unlike `cin_actr.h` (which handles motion/behavior), `cin_evnt.h` handles *declarative state changes* (palette, sprite visibility, etc.). Events are often triggered by or trigger actor state changes, but are kept separate.
- **Global queue head/tail pointers**: Avoids allocation of a container struct; minimal overhead for a single queue.

## Data Flow Through This File
1. **Load phase**: `ParseEvent()` reads cinematic script data, calls `CreateEvent(time, type)` → `AddEvent()` → event lands in `firstevent`/`lastevent` queue (likely time-sorted)
2. **Playback phase**: Per-frame, `UpdateCinematicEvents(current_time)` iterates queue, executes all events where `event.time <= current_time`
3. **Event execution**: Event handler (palette change, sprite animation, FLI frame, etc.) applies visual state change
4. **Cleanup**: When cinematic ends or is aborted, `ShutdownEvents()` deallocates queue and resets globals
5. **Manual removal**: `DeleteEvent()` allows on-demand removal (e.g., if cinematic is aborted mid-playback)

## Learning Notes
- **1990s game engine idiom**: Time-keyed event queues were the standard before message buses or signal systems. Still efficient and simple.
- **Contrast with modern patterns**: Modern engines often use event emitters or ECS message stores; this uses a raw linked list with iteration. No callbacks—events are data, handlers are separate.
- **Script-driven cinematics**: The presence of `ParseEvent()` and `PrecacheCinematic()` suggests cinematics are authored as *data* (scripts) rather than code, which is a best practice even today.
- **Asynchronous effect handling**: Events decouple *when* an effect is scheduled from *how* it's rendered, allowing complex multi-effect sequences without tight coupling.

## Potential Issues
- **No apparent priority/ordering**: Events at the same `time` value are executed in insertion order (or queue order). If order matters, there's no explicit priority field.
- **Linear iteration in `UpdateCinematicEvents()`**: If there are many events and cinematics are long, O(n) iteration per frame could become expensive. No visible early-exit or binary search optimization.
- **Global queue state**: No thread-safety or reentrancy guards visible. Assumes single-threaded cinematic playback (typical for 1990s games).
