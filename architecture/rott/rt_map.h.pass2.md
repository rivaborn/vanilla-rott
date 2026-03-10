# rott/rt_map.h — Enhanced Analysis

## Architectural Role
Public interface for the map subsystem, exposing in-game map visualization (both normal and cheat-activated modes). This file bridges the game loop/rendering pipeline (via `DoMap`) with the debug/cheat code handler (`CheatMap`), indicating map rendering is treated as both a gameplay feature and a developer tool. The minimal nature of this header suggests map functionality is isolated into its own rendering module.

## Key Cross-References

### Incoming (who depends on this file)
- **Game loop / main renderer**: Calls `DoMap(x, y)` during each frame to render map HUD/overlay at screen position
- **Debug/cheat command processor**: Calls `CheatMap()` when cheat code is entered (likely from input handler or cheat dispatcher)
- *Note: Specific callers not visible in provided cross-reference excerpt, but function names and signatures infer these roles*

### Outgoing (what this file depends on)
- **Implementation in `rt_map.c`**: Implementation visible to be calling `ChangeMapScale` (also in rt_map.c, suggesting internal map state management)
- **Related utility**: `FixMapSeen` exists in rt_map.c, suggesting map visibility/fog-of-war state tracking
- **No external includes in header**: Map module is self-contained; dependencies are internal to rt_map.c

## Design Patterns & Rationale

- **Minimal Public API**: Only two functions exported—reflects 90s game engine design where modules expose just the essential interface
- **Cheat as First-Class Feature**: `CheatMap()` is not hidden or namespaced separately; cheat codes are part of the normal build, not retrofitted
- **Void return + side effects**: Both functions use void returns, indicating they operate on global/static map state (common pattern for immediate mode rendering in software rasterizers)
- **Coordinate-based rendering**: `DoMap(x, y)` takes explicit screen coordinates, suggesting it renders a map HUD/overlay at a specific position rather than full-screen

## Data Flow Through This File

1. **Normal flow**: Main game loop → `DoMap(screen_x, screen_y)` → renders current map state to framebuffer at given position
2. **Cheat activation**: Player enters cheat code → input handler → `CheatMap()` → toggles map visibility or reveals fog-of-war → next `DoMap` call reflects change
3. **Internal state**: Map visibility flags, scale, and "seen" tiles are maintained in rt_map.c statics; both functions read/write this state

## Learning Notes

- **No type safety in interface**: Parameters are bare `int`; semantics (pixel vs. tile coords, range constraints) must be known by caller—typical of C89 era code
- **Immediate-mode style**: Void functions with side effects on global state, not functional or ECS-style (contrast with modern game engines' systems architecture)
- **Cheat culture**: Cheat codes are engineered features, not emergent bugs; `CheatMap` sits alongside normal gameplay, suggesting Apogee valued player experience and debugging
- **Modular rendering**: Map visualization is a separate concern from game logic, enabling independent iteration on HUD design

## Potential Issues

- **Coordinate semantics undefined**: Without reading rt_map.c, callers must infer whether (x, y) are screen pixels, tile indices, or world coordinates
- **No validation**: No bounds checking visible; out-of-range coordinates could cause buffer overruns in implementation
- **Global state coupling**: Both functions operate on hidden statics in rt_map.c; no state encapsulation means map state is tightly coupled to the renderer
