# rott/_rt_film.h — Enhanced Analysis

## Architectural Role
This header is the **type definition backbone** for ROTT's timeline-based cutscene/demo playback system. It sits between the high-level cinematic scripting layer (`cin_*.c`) and the lower-level runtime animation system (`rt_film.c`). The `event` and `actortype` structures provide the contract for sequencing timed visual changes—backgrounds, sprites, palette swaps, fades—during non-interactive sequences. This design separates **what to show and when** (data, defined here) from **how to show it** (rendering, handled elsewhere).

## Key Cross-References

### Incoming (who depends on this file)
- **`cin_main.c`** — Likely parses cinematic script files and populates event arrays using structures defined here
- **`cin_actr.c`** (`AddCinematicActor`) — Creates and manages actor instances during cinematic playback; the `actortype` struct directly models actor runtime state
- **`cin_evnt.c`** (`AddEvent`, `CreateEvent`) — Event-creation API; translates script directives into `event` struct instances  
- **`rt_film.c`** (`AddEvents` function) — Implements film-level management using these types; loads and sequences events

### Outgoing (what this file depends on)
- **Standard C types only** — Header is self-contained; no external subsystem dependencies (by design, isolation principle)

## Design Patterns & Rationale

**Timeline/Keyframe Animation:**  
Events are timestamped (`time` field) and sorted chronologically. Actors iterate through events as tics elapse, interpolating position/scale via velocity deltas (`dx`, `dy`, `dscale`). This is idiomatic 1990s demo/cutscene design: simple, cache-friendly, and controllable frame-by-frame.

**Capacity Limits as Constants:**  
`MAXEVENTS = 100` and `MAXFILMACTORS = 30` are hard-coded because:
- This was pre-dynamic-allocation era
- Fixed limits simplified memory management and prevented unbounded growth
- Scripts were authored with these constraints in mind

**Separation of Concerns:**  
The header defines **types only**—no init/update/render logic. Implementation (`rt_film.c`) handles the mechanics; cinematic scripting (`cin_*.c`) handles authoring. This loose coupling allowed artists and programmers to work in parallel.

## Data Flow Through This File

```
Script file
    ↓
cin_main.c parses & calls AddEvents (rt_film.c)
    ↓
AddEvents populates event[MAXEVENTS] array
    ↓
Cinema playback loop:
  - Actor state (actortype) tracks current event index & elapsed tics
  - Each frame: interpolate (curx, cury, curscale) from event data
  - Render backdrop/sprite at interpolated position/scale
  - On time boundary, advance actor to next event
    ↓
Result: smooth, deterministic cutscene replay
```

## Learning Notes

**Idiomatic to this era (early-to-mid 1990s):**
- **No ECS/component systems** — Actors are pure data structs with fields for position/scale state
- **Declarative timeline** — Events are immutable once created; playback simply reads and interpolates
- **No velocity abstraction** — Velocity stored as raw `dx`, `dy`, `dscale` deltas; no separate physics object
- **String asset names** — `char name[10]` references assets (backgrounds, sprites) by name, likely resolved at load time

**Contrast with modern engines:**  
Today you'd use animation tracks/curves, blend trees, or timeline editors (e.g., Sequencer in UE5). ROTT's approach is simpler but less expressive—no easing functions, no hierarchical transforms, no event callbacks mid-timeline.

## Potential Issues

1. **Fixed-size name field** — `char name[10]` is a hard limit; long asset names truncate silently.
2. **No explicit interpolation rule** — How is (curx, cury) actually computed from (x, y, dx, dy) over a frame? Logic must live in `rt_film.c`; unclear from types alone.
3. **No viewport/bounds checking** — Sprites can move off-screen undetected; no clipping constraints in the data model.
4. **Scale discontinuities** — If `dscale` steps unevenly, scaling may jitter; no smoothing guarantees.

---

**Sources:** Cross-references show `cin_actr.c`, `cin_evnt.c`, `cin_main.c`, and `rt_film.c` as the primary consumers. The cinematic subsystem (cin_*) is distinct from the runtime actor/AI system (rt_actor.c).
