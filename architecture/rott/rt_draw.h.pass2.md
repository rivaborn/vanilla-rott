Looking at the file content, first-pass analysis, and cross-reference data, I'll produce the enhanced second-pass analysis:

---

# rott/rt_draw.h — Enhanced Analysis

## Architectural Role

`rt_draw.h` is the **public interface to the core 3D rendering subsystem**, serving as the contract between the game loop and the renderer. It exposes ray-traced visibility culling, visible object management, camera state, and screen compositing. The file bridges high-level game logic (via cinematic and UI functions) with low-level frame rendering (`ThreeDRefresh`), making it a central hub that coordinates actor rendering (through `vislist`), map geometry (via `tilemap` and `spotvis`), and dynamic lighting. The visible object list (`vislist`) acts as the primary communication channel between physics/actor updates and final rendering.

## Key Cross-References

### Incoming (who depends on this file)
- **Game loop / main engine** calls `ThreeDRefresh()` once per frame and `CalcTics()` for timing
- **Screen/UI subsystems** call title screen functions (`ApogeeTitle`, `DopefishTitle`, `DoEndCinematic`, `DoCreditScreen`, `DoLoadGameSequence`) — UI code lives above the rendering layer
- **Cinematic system** (`cin_*.c` files) calls `DoInBetweenCinematic()` for in-game cutscenes with screen overlays
- **Actor/collision systems** populate `vislist` indirectly; `ThreeDRefresh` determines which actors are visible
- **Screen effects** consume `RotateBuffer()` for wipes/transitions during cinematics
- **Map/door system** relies on `tilemap` and `spotvis` arrays being up-to-date

### Outgoing (what this file depends on)
- **Math lookup system** (`lookups.c`): calls `BuildTables()` to populate `sintable`, `costable`, `tantable` at startup
- **View calculation subsystem** (`rt_view.c`): implicitly updates camera state (`viewx`, `viewy`, `viewangle`, `viewsin`, `viewcos`) that this header declares
- **Static/wall rendering** (`rt_stat.c`): reads `tilemap[x][y]` for wall geometry and `spotvis` for visibility state
- **Visibility/culling system**: uses `mapseen` to track explored areas and `lights` array for dynamic light intensity lookup
- **Actor system** (`rt_actor.c`, `rt_playr.c`): actors don't directly call render functions; instead, their positions are read by `ThreeDRefresh` to populate `vislist`
- **Frame buffer / display driver**: `FlipPage()` is the final output stage to hardware

## Design Patterns & Rationale

**Immediate-mode rendering with cached visibility:** The engine uses a **classic early-90s approach** of precomputing math tables (sin/cos/tan) once at startup, then iteratively building a sorted visible object list per frame. This avoids per-frame division (slow on 386/486) and dynamic allocation.

**Global state for camera and rendering:** `viewx`, `viewy`, `viewangle` plus precomputed `viewsin`/`viewcos` are global. This is typical for single-camera, arcade-style games and simplifies the render loop, but prevents easy parallelization or multi-camera effects.

**Visible object list as a data pump:** `vislist[]` + the iteration pointers (`visptr`, `visstep`, `farthest`) form a **sorted intermediate representation**. Actors are not drawn directly; they're culled into this list, then rasterized in back-to-front order. This decouples physics from rendering.

**Light source grid:** The `lights` array uses a spatial grid (indexed via the `LightSourceAt` macro: `lights + (x<<7) + y`), allowing O(1) light lookup during wall rasterization. The `<<7` suggests 128×128 map tiles.

**Direction angle tables:** Pre-built `dirangle8` and `dirangle16` arrays avoid angle quantization math in hot paths (enemy AI, player input).

## Data Flow Through This File

1. **Input:** Each frame, the game loop has updated:
   - Camera state (placed in global `viewx`, `viewy`, `viewangle` by game/physics code)
   - Actor positions (read by `ThreeDRefresh` from actor list, not passed directly)
   - Tilemap (static, updated only on door state changes or level load)

2. **Transformation:**
   - `ThreeDRefresh()` ray-traces from camera through each screen pixel, finding wall intersections (`xintercept`, `yintercept`)
   - For each hit, it queries the light grid and visibility cache (`spotvis`, `mapseen`)
   - Visible actors are collected into `vislist` and sorted by depth
   - `CalcHeight()` converts 3D depth to 2D screen height using precomputed sin/cos

3. **Output:**
   - Screen buffer is filled with rasterized walls and sprites (colors looked up via `shadingtable` and `colormap` per `visobj_t`)
   - `FlipPage()` swaps to display
   - UI/cinematic overlays render on top (title/credit screens, in-between cinematics)

## Learning Notes

**Idiomatic 1990s console/arcade game engine design:**
- Precomputed lookup tables eliminate per-frame expensive operations (trig, division)
- Global camera state over scene graphs (simpler, faster for single-view games)
- Immediate-mode rendering (draw each frame from scratch) vs. retained-mode (scene graph)
- Visible object list as the primary render interface (not a draw call API)

**Early raycasting era (Wolfenstein 3D legacy):** ROTT extended the classic raycasting engine (straight rays per vertical strip) with full 3D actor sprites and lighting, but retained the ray-per-column architecture and flat light grid.

**Lighting model:** The `lights` array + `LightSourceAt` macro provide **grid-based dynamic lighting**. Each tile can have a light level; walls sample this during rasterization. Far simpler than pixel-perfect light calculations, but sufficient for 320×240 software rendering.

**Director angles in AI/control:** The `dirangle8` and `dirangle16` lookups suggest actors quantize direction to 8 or 16 cardinal directions, likely to avoid expensive atan2 calls in pathfinding and enemy targeting.

## Potential Issues

- **Global state coupling:** All render state is global, making it impossible to render to multiple viewports or run parallel render passes without careful synchronization
- **Fixed grid resolution:** The light grid (`(x<<7)`) and the visible object limit (`MAXVISIBLE=256`) are compile-time constants; dynamic scaling (e.g., for higher resolution) would require refactoring
- **No explicit resource lifecycle:** `shadingtable` and `lights` pointers are declared but not initialized here; callers must ensure they point to valid memory before `ThreeDRefresh()` runs

---
