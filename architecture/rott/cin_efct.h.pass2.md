# rott/cin_efct.h — Enhanced Analysis

## Architectural Role
This file defines the **public API for cinematic effect composition and rendering**—the primary abstraction layer through which higher-level cinematic scripting systems (cin_main.c, cin_evnt.c) control low-level animation, sprite, background, and palette effects. It sits at the boundary between **event management/queueing** (from cin_evnt.h) and **frame-level rendering**, serving as the polymorphic dispatcher that allows generic event handlers to route to type-specific implementations without knowing concrete event structures.

## Key Cross-References

### Incoming (who depends on this file)
- **cin_main.c**: Cinematic execution loop likely calls Draw*/Update* functions in sequence per frame
- **cin_evnt.c / cin_actr.c**: Event creation/queueing systems invoke Spawn* factory functions to populate the event queue
- **cin_glob.h dependency chain**: Timing synchronization (CinematicDelay) and abort polling drive the frame loop that cycles through these Draw/Update calls

### Outgoing (what this file depends on)
- **cin_glob.h**: Provides `CinematicDelay()`, `CinematicAbort()`, `GetCinematicTime()` for frame-sync and event loop control
- **cin_def.h**: Type definitions (flicevent, spriteevent, backevent, paletteevent, enum_eventtype)
- **Graphics subsystem** (not visible in cross-ref): Each Draw* function writes directly to frame buffer/palette
- **WAD resource system**: `DrawPostPic(lumpnum)` loads raw image lumps by index
- **FLIC decoder**: Underlying implementation (cin_flic.c implied, not in cross-ref excerpt)

## Design Patterns & Rationale

**Factory Pattern** (Spawn* functions)  
Each Spawn* function allocates and initializes a typed event, then likely calls `AddEvent()` (from cin_evnt.h) internally. This decouples script parsing from event object construction.

**Type-Agnostic Dispatch**  
`DrawCinematicEffect(enum_eventtype, void*)` and `UpdateCinematicEffect()` are virtual dispatch points—the void* pointer is cast to the appropriate type based on the enum. This is a **pre-C++ inheritance workaround**: a single event queue can hold mixed types without requiring knowledge of concrete types at iteration time. The return boolean signals completion (used to dequeue finished effects).

**Immutable Spawn Parameters**  
Sprite and background Spawn calls accept start and end values (x, y, scale, etc.). The implementation computes per-frame deltas (dx, dy, dscale) at allocation time. Update functions then apply these deltas linearly. This is **pre-computation for real-time efficiency**—critical on 1990s hardware.

## Data Flow Through This File

1. **Setup Phase** (script parsing in cin_main.c)
   - Script reader parses cinematic definition
   - Calls Spawn* functions with animation parameters
   - Each factory allocates struct, computes derivatives, queues via AddEvent()

2. **Playback Loop** (main loop in cin_main.c)
   ```
   while (!CinematicAbort()) {
     for each queued event:
       UpdateCinematicEffect(type, effect)  // Advance frame/position
       DrawCinematicEffect(type, effect)    // Render
     DrawClearBuffer() / DrawBlankScreen()  // Frame prep/cleanup
     CinematicDelay()                       // Sync timing
   }
   ```

3. **Completion**
   - Update functions return false when duration expires
   - Event dequeued; memory freed (implementation detail in cin_evnt.c)

## Learning Notes

**1990s Real-Time Graphics Pattern**  
This system demonstrates typical late-90s game architecture:
- **No retained-mode scene graph** (modern engines use ECS or node hierarchies)
- **Immediate-mode rendering** (Draw* called once per frame with live data)
- **Pre-computed interpolation** (derivatives baked at spawn time, not evaluated per-frame—saves CPU)
- **Polymorphism via enums + void*** (no virtual tables; enum dispatch is faster on CPU-cache-limited hardware)

**Separation of Animation & Rendering**  
Spawn* functions handle all parameter validation and data setup; Draw* functions are "dumb" and just copy precomputed state to screen. This is **good separation of concerns**, but also reflects the absence of a unified resource manager—each effect type must manually load its own textures.

**Palette Swapping as Discrete Event**  
`SpawnCinematicPalette()` and `DrawPalette()` treat palette changes as first-class timeline events, not as properties of other effects. This reflects VGA-era graphics where palette is global state, changed via hardware register writes—not per-pixel like modern 24-bit rendering.

## Potential Issues

1. **Type Safety**: Void pointer dispatch relies on caller passing the correct enum-to-pointer type correspondence. Mismatches cause silent memory corruption.
2. **No Visible Resource Cleanup**: Spawn functions allocate memory; deallocation must occur in Update when returning false. If the event queue is manually cleared (error case), leaks occur.
3. **Linear Interpolation Only**: All motion and scaling use simple `value += delta` per frame. No easing curves, acceleration, or ease-in/out—fine for mechanical cutscenes, but inflexible.
4. **Global Palette State**: `DrawPalette()` and `CinematicSetPalette()` coordinate palette changes; if effects are interrupted mid-cinematic, palette may not restore to previous state.
