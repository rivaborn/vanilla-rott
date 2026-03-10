# rott/cin_glob.c — Enhanced Analysis

## Architectural Role
This file provides the cinematic subsystem's primary interface to engine timing and input services. It acts as an abstraction facade between high-level cinematic playback logic (in `cin_main.c`, `cin_actr.c`, etc.) and low-level engine primitives (`rt_draw.c` tick system, `rt_in.c` input handling). This decoupling allows cinematics to operate independently of rendering and input implementation details.

## Key Cross-References

### Incoming (who depends on this file)
- **Cinematic playback loop** (`cin_main.c`, inferred from first-pass): calls `CinematicDelay()` once per frame and `CinematicAbort()`/`ClearCinematicAbort()` for skip detection
- **Title/end sequence rendering** (`rt_draw.c`: `ApogeeTitle()`, `DopefishTitle()`, `DoEndCinematic()`, per first-pass): likely uses these functions during rendering
- **Cinematic event system** (`cin_evnt.c`, `cin_evnt.h`): may coordinate timing with events

### Outgoing (what this file depends on)
- **`rt_draw.c`**: `CalcTics()` (advances tick counter), reads `ticcount` (global extern) for elapsed time
- **`rt_in.c`**: `IN_CheckAck()` (polls acknowledgment/skip input), `IN_StartAck()` (clears input state)
- **Cinematic utility layer** (`cin_util.c`, `cin_util.h`): separate module handles palette/graphics; this module handles timing/input only

## Design Patterns & Rationale

**Facade Pattern**: The file wraps two distinct engine subsystems (timing in `rt_draw.c`, input in `rt_in.c`) under a cinematic-specific API. This shields cinematic code from engine-level details and establishes a clear contract.

**Tick-based timing over real-time**: Uses `ticcount` (discrete frame ticks) rather than high-resolution timers. This is deterministic and synchronizes cinematics with the main game loop, essential for networked play and demo playback (common in 1990s engines).

**Stateless query API**: All functions are reads or simple state resets—no cinematic state is managed here. The module is purely a pass-through, leaving state management to `cin_main.c` and related files.

## Data Flow Through This File

```
rt_draw.c (CalcTics, ticcount)
    ↓
CinematicDelay() [advances ticcount via CalcTics()]
CinematicGetTime() [reads ticcount]
    ↓
Cinematic playback loop (cin_main.c, etc.)
    ↓
CinematicAbort() [checks rt_in.c]
ClearCinematicAbort() [resets rt_in.c]
    ↓
rt_in.c (IN_CheckAck, IN_StartAck)
```

Timing and input are separate streams that converge in the cinematic loop: each frame, timing advances; each frame, input is polled and cleared if consumed.

## Learning Notes

**Era-specific design**: The tick-based model reflects 1990s engine architecture, where determinism and network synchronization were paramount. Modern engines typically use elapsed delta-time (floats) and centralized input handlers.

**Minimal abstraction**: The file is intentionally thin—only 4 functions, each a single-line wrapper. This is idiomatic for the era; more sophisticated abstraction (e.g., cinematic state machines, frame callbacks) lives in `cin_main.c`.

**Input acknowledgment pattern**: The `CheckAck()`/`StartAck()` model (not `GetInput()` or similar) suggests a simple, blocking input scheme—wait for any key to skip. More complex input (menus, multiple keys) likely lives in `rt_in.c`.

**Separation of concerns**: Graphics (`cin_util.c` for palettes), timing (`cin_glob.c`), events (`cin_evnt.c`), and actors (`cin_actr.c`) are split across files, suggesting a modular cinematic subsystem design.

## Potential Issues

None clearly inferable. The simplicity of this module makes it unlikely to contain bugs. The only implicit assumption is that `ticcount` is properly maintained by `rt_draw.c` and that `IN_CheckAck()`/`IN_StartAck()` are correctly paired—mismatched calls could cause input to stick or skip.
