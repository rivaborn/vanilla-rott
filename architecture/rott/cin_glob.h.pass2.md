I can see from the cross-reference context that the cinematic functions are defined in both the header and implementation file. However, the cross-reference excerpt provided only shows the function definition map, not the complete call graph. Let me write the second-pass enhanced analysis with the insights available:

---

# rott/cin_glob.h — Enhanced Analysis

## Architectural Role
This header sits at the **timing synchronization boundary** between the cinematic playback system (cin_actr.h, cin_evnt.h, cin_main.c) and the interrupt-driven display loop. It decouples VBlank hardware timing from cinematic logic by providing a stable frame-rate API. The 35 Hz `CLOCKSPEED` macro ensures cinematics play at a fixed rate independent of actual rendering performance—essential for lip-sync, animation synchronization, and deterministic replay.

## Key Cross-References
### Incoming (who depends on this file)
- **cin_main.c**, **cin_actr.c**, **cin_evnt.c** — The cinematic subsystem (playback, actor animation, event scheduling) calls `CinematicDelay()` in their frame loops and checks `CinematicAbort()` for user skip input
- **rt_film.c** — References `AddEvents()` which likely orchestrates cinematic events; needs timing from `GetCinematicTime()`
- **Input handling** (rt_playr.c or rt_main.c) — Sets the abort flag (cleared via `ClearCinematicAbort()`)

### Outgoing (what this file depends on)
- **isr.h** / ISR subsystem — Reads/writes `VBLCOUNTER` and `ticcount` (interrupt counter state); `CinematicDelay()` blocks until the ISR advances the counter
- **rt_def.h** — Provides `boolean` type; no direct state dependencies visible
- **rt_util.h** — Included but purpose opaque from header alone (likely general utility macros)
- **<time.h>** — Included but not used in declarations; likely for internal millisecond conversion in .c file

## Design Patterns & Rationale
**Polling + Blocking Synchronization**: Rather than callbacks or events, cinematics use busy-wait (VBlank polling). This is typical of early 1990s DOS/console engines where:
- ISR increments a volatile counter at fixed intervals
- `CinematicDelay()` spins until counter advances (or until a deadline)
- Abort flag enables graceful user cancellation without interrupting playback

**Separation of Concerns**: Timing is abstracted away from cinematic content (actors, events). This allows the cinematic engine to assume a stable 35 Hz frame rate without managing hardware details. The `CLOCKSPEED` macro provides a single point of configuration.

## Data Flow Through This File
```
Input:
  - ISR VBlank interrupt updates VBLCOUNTER
  - User input sets abort flag (elsewhere)

Transform:
  - CinematicDelay(): block until next frame tick
  - GetCinematicTime(): read elapsed ticks since cinematic start
  - CinematicAbort()/ClearCinematicAbort(): query/reset abort flag

Output:
  - Cinematic subsystem advances its frame loop in lockstep with display refresh
  - Audio/animation synchronizes to elapsed time from GetCinematicTime()
  - User skip request propagates upward to cinematic controller
```

## Learning Notes
**1990s game engine pattern**: Hard synchronization to display (VBlank) was essential before triple-buffering and vsync APIs. This design assumes:
- Single-threaded execution
- ISR updates volatile counter at known interval
- Game loop polls that counter

**Modern equivalents**: Today, engines use frame deltas, time-stepping, or render-thread callbacks. This code is fundamentally tied to DOS interrupt architecture.

**Cinematic subsystem architecture**: The presence of `cin_actr.h`, `cin_evnt.h`, `cin_util.h`, and `cin_glob.h` suggests a modular cutscene system:
- **Actors** — animated sprites/objects in the cinematic
- **Events** — scripted sequences (camera moves, sound cues)
- **Globals** — timing and control (this file)
- **Utils** — palette transitions, graphics setup

This mirrors modern game engine cinematics tooling, but implemented entirely in C with explicit state management rather than a scene graph or timeline editor.

## Potential Issues
- **Race condition**: `CinematicAbort()` reads a flag set by input code; if input runs in ISR context, there's no memory barrier. The `boolean` type likely maps to `volatile` in rt_def.h, but this should be verified.
- **Busy-wait latency**: `CinematicDelay()` blocks the entire engine. If timing is tight, frame drops or audio stuttering could occur. No visible backpressure mechanism if rendering falls behind.
- **No frame-skip logic**: If a frame takes longer than 1/35th second, `GetCinematicTime()` may not advance, causing audio to desync. No detection of this condition visible in header.

---

**Note**: Full cross-call analysis would require seeing actual call sites in .c files. The provided cross-reference index shows function definitions but not invocation sites. To verify the above, check `cin_main.c`, `cin_actr.c`, and `rt_playr.c` for actual usage patterns.
