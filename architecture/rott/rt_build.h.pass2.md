# rott/rt_build.h — Enhanced Analysis

## Architectural Role

`rt_build.h` defines the **menu buffer compositing subsystem**—an off-screen rendering layer that allows the UI/menu system to draw complex scenes (text, shapes, pictures) independently of the main game viewport, then composite them onto screen with per-frame positioning and transparency effects. This is central to ROTT's UI architecture: menus need smooth animation, layering, and fade effects that would be difficult to achieve via immediate-mode rendering. The buffer acts as a staging area between the menu state machine (in `rt_menu.c` or equivalent) and the final screen output, enabling animated menu transitions and intensity-based lighting effects.

## Key Cross-References

### Incoming (Who Calls This)
- Menu system implementation (likely `rt_menu.c`, `rt_game.c` – though exact callers not visible in provided cross-reference excerpt)
- Any UI layer that needs animated, composited display of shapes and text
- Setup/shutdown called from engine initialization (`rt_main.c` equivalent)

### Outgoing (What This Depends On)
- **Lookup table globals**: `intensitytable` (intensity mapping for transparency/lighting effects)
- **Control flow state**: `Menuflipspeed` (external configuration for animation timing)
- **Underlying graphics subsystem**: The implementation (`rt_build.c`) likely calls lower-level video/pixel operations (not visible in this header, but typical engine dependency)
- Shape/sprite asset loading system (implied by `shapenum` parameters)

## Design Patterns & Rationale

**Double-Buffering for Smooth Animation**  
The `ClearMenuBuf()` → `Draw*()` → `RefreshMenuBuf()` → `FlipMenuBuf()` pattern implements classic double-buffering: one buffer accumulates frame commands, then atomically swaps to screen. This prevents flicker and allows animated menu rotation/positioning without tearing.

**Lookup-Table-Based Effects (Intensity)**  
The `intensitytable` global is a classic 1990s optimization: rather than compute transparency per-pixel in real-time, precompute a 256-entry lookup that maps (original_pixel, intensity_level) → final_pixel. This enables efficient intensity-based lighting and transparency on 256-color palettes—standard for DOS/early 90s games before true-color graphics.

**Function Overloading via Naming Convention**  
Multiple variants (`DrawMenuBufItem`, `DrawTMenuBufItem`, `DrawIMenuBufItem`, `DrawColoredMenuBufItem`) encode *which effect to apply* in the function name rather than using an enum parameter. This is typical of C codebase patterns where dispatch happens at compile-time and inlining is desirable.

**Polar Coordinate Positioning**  
`PositionMenuBuf(angle, distance, ...)` uses polar coords rather than Cartesian—suggesting menus can rotate around a center point or emerge from screen edges. Fits the mid-90s aesthetic of rotating, zoomed menu screens (common in id-engine games).

## Data Flow Through This File

1. **Initialization Phase**: `SetupMenuBuf()` allocates the off-screen buffer in system memory (likely VRAM or conventional RAM).
2. **Per-Frame Draw Phase**:
   - `ClearMenuBuf()` resets to blank
   - Series of `Draw*()` calls write shapes, text, pictures into the buffer
   - `RefreshMenuBuf(time)` updates animation state (menu rotation, alpha blending based on elapsed time)
3. **Display Phase**: `FlipMenuBuf()` copies/composites the menu buffer to the visible screen, possibly with positioning transforms applied by `PositionMenuBuf()`
4. **Shutdown**: `ShutdownMenuBuf()` deallocates resources.

The `intensitytable` is sampled during drawing; `Menuflipspeed` gates animation rate.

## Learning Notes

**Idiomatic to 1990s Game Engines**
- **Lookup-table graphics effects**: Modern engines use shaders; ROTT pre-bakes effects into lookup tables for 256-color palette constraints.
- **Double-buffering as explicit API**: Modern engines hide buffering internally; here it's exposed (`ClearMenuBuf`, `FlipMenuBuf`) because the menu system needs explicit control over frame timing and composition.
- **Shape/sprite IDs as integers**: No asset management abstraction—shapes are referenced by raw numeric ID, requiring caller knowledge of the numbering scheme.

**Game Engine Concepts**
- This is a simple **render target abstraction** (off-screen buffer) before GPU-based framebuffer objects became standard.
- The variety of drawing functions (`DrawTMenuBufItem`, `DrawIMenuBufItem`, etc.) mirrors pixel blitting modes in legacy graphics libraries (e.g., transparency, intensity, color replacement).
- Polar-coordinate menu positioning hints at **viewport transforms** predating modern 3D engine camera abstractions.

## Potential Issues

1. **No visible ownership of `intensitytable`**: Where is it allocated, who owns its lifetime? If it's statically allocated, there's tight coupling to that assumption.
2. **No bounds checking visible**: The header doesn't show assertions on `shapenum`, coordinates, or buffer dimensions—classic 1990s code assumes valid input from trusted menu system.
3. **Global state (`Menuflipspeed`, `intensitytable`)**: Hard-coded globals make the menu buffer non-reentrant or multi-instance-hostile, if that were ever needed.
4. **String rendering variants**: Multiple `MenuBufPrint*` functions with similar names but different behavior (centered, shaded, line-wrapped) invite misuse; no enum-based dispatch.
