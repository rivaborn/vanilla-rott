Looking at the file content, first-pass analysis, and cross-references, I'll write the enhanced second-pass analysis:

---

# rott/cin_main.c — Enhanced Analysis

## Architectural Role
`cin_main.c` is the **cinematic orchestration layer**—a state machine that coordinates the loading, parsing, and real-time execution of cinematic scripts. It sits at the intersection of the script subsystem (scriplib), the cinematic subsystems (actors, events, effects in cin_actr/cin_evnt/cin_efct), and resource management (WAD loader). The file acts as the bridge between the game's high-level flow (which calls `PlayMovie()`) and the lower-level cinematic presentation engine, managing timing and frame synchronization critical for synchronized animation and audio playback.

## Key Cross-References
### Incoming (who depends on this file)
- **`PlayMovie()`** is the public API called from game flow control (likely `rt_game.c`, `rt_main.c`, or intro sequence handlers) to play named cinematics
- Global `cinematicdone` is read/written by playback loop and abort handlers (from `cin_glob.h` functions like `CinematicAbort()`)
- Cinematic subsystems (cin_actr, cin_evnt, cin_glob) expose callbacks that this file invokes during playback

### Outgoing (what this file depends on)
- **Script subsystem** (`scriplib.h`): `GetToken()`, `ParseNum()`, global parsing state (`script_p`, `scriptend_p`, `endofscript`)
- **Resource loading** (`w_wad.h`): `W_GetNumForName()`, `W_CacheLumpNum()`, `W_LumpLength()` for WAD-based script loading
- **Cinematic subsystems**: `StartupEvents()`, `ShutdownEvents()`, `UpdateCinematicEvents()`, `StartupCinematicActors()`, `UpdateCinematicActors()`, `DrawCinematicActors()`, `ShutdownCinematicActors()`, `PrecacheCinematic()`
- **Timing** (`cin_glob.h`): `GetCinematicTime()`, `CinematicAbort()`, `ClearCinematicAbort()` 
- **Graphics** (`modexlib.h`, `lumpy.h`): `ProfileDisplay()` for performance profiling
- **Disk I/O** (`scriplib.h`): `LoadScriptFile()` for non-WAD cinematic scripts

## Design Patterns & Rationale
**State Machine + Initialization Template**: The classic game engine pattern—`Startup()` → `Main Loop()` → `Shutdown()`. This ensures subsystem init order, clean resource release, and decoupling of setup from execution logic.

**Dual Loading Modes**: `uselumpy` boolean switches between disk-based (`LoadScriptFile`) and WAD-cached (`CacheScriptFile`) loading. This reflects DOS-era pragmatism: during development, iterate quickly off disk; in production, ship with WAD lumps for faster, safer loading and lower disk footprint.

**Profiling-Based Timing**: `ProfileMachine()` measures rendering overhead once at startup, then uses that constant (`profiletics`) for all frame deltas. This avoids frame-to-frame timer variance and ensures synchronized playback despite variable system load—critical for lips sync and music sync in cinematics. It trades accuracy for consistency.

**Accumulated Time Model**: `ParseCinematicScript()` accumulates relative time deltas and dispatches events at absolute times. The inner loop increments `cinematictime` once per tic, allowing actors/events to reason about linear time progression rather than frame boundaries.

## Data Flow Through This File
1. **Load Phase**: `PlayMovie()` → `GrabCinematicScript()` → chooses `LoadScriptFile()` (disk) or `CacheScriptFile()` (WAD)
2. **Parse Phase**: `ParseCinematicScript()` tokenizes and accumulates time values, calls `ParseEvent()` to register time-keyed events into cin_evnt subsystem
3. **Precache Phase**: `PrecacheCinematic()` (defined elsewhere, called from `PlayMovie()`) pre-loads graphics/audio assets
4. **Execution Phase**: Main loop: 
   - `GetCinematicTics()` waits for timer to advance, returns frame delta
   - For each tic in frame: `UpdateCinematicEvents(time)`, `UpdateCinematicActors()` mutate actor state
   - `DrawCinematicActors()` renders all actors to screen
   - Loop until `CinematicAbort()` signals end or user interrupt
5. **Shutdown**: Release events, actors, free cached memory

## Learning Notes
**Tics vs. Frames**: This engine separates logical "tics" (game timesteps) from rendering frames. A tic is a logical update; a frame may contain multiple tics or vice versa. Modern engines typically unify these, but separating them allows deterministic replay and flexible frame skipping.

**DOS-Era Pragmatism**: The profiling hack, busy-wait in `GetCinematicTics()`, and dual loading paths reflect 1990s constraints: CPU variability, no OS task scheduling guarantees, and storage bandwidth limits. Modern engines use high-resolution timers and async I/O instead.

**No Event Queue Visible Here**: Events are dispatched directly into cin_evnt subsystem during parse. The actual event execution happens during the main loop via `UpdateCinematicEvents(time)`. This defers event execution but pre-parses all events—useful for scripted timing but limits dynamic events.

**Global Timing State**: The static variables (`cinematictime`, `cinematictics`, `cinematictictime`) are tightly coupled; modern engines would wrap these in a struct and pass by reference. The tight coupling here reflects the age of the codebase and DOS conventions.

## Potential Issues
- **Busy-Wait in `GetCinematicTics()`**: The loop `while (time==cinematictictime)` is CPU-intensive and unresponsive. On modern hardware it would waste power; on DOS it could interfere with interrupt-driven audio.
- **No Error Handling**: Script load failures, missing WAD lumps, or bad script syntax return silently. A real script parse error would crash in `ParseEvent()` if it dereferences bad tokens.
- **Single Profiling Run**: If the machine slows down mid-cinematic (e.g., disk thrashing), `profiletics` is stale and frames may skip or judder.
