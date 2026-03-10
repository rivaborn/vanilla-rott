# rott/rt_view.h — Enhanced Analysis

## Architectural Role
This header is the **view/rendering subsystem facade** that bridges the game engine's raycasting 3D renderer with the VGA display layer. It manages the **camera projection parameters** (focal width, viewport size, aspect ratios) and the **dynamic lighting pipeline** (illumination levels, gamma correction, darkness/shade tables). The globals declared here are read constantly during rendering: `pixelangle[]` is a lookup table used per ray during raycasting, while the shading globals (`normalshade`, `maxshade`, `minshade`) are applied per wall/sprite during scanline fill. Together with `modexlib.h`, this file forms the contract between high-level game logic (which calls `SetViewSize()`, `UpdateLightLevel()`) and the low-level rendering code (which reads `pixelangle[]`, `colormap`, shading bounds).

## Key Cross-References

### Incoming (who depends on this file)
Based on the codebase structure and cross-reference index:
- **rt_draw.c / rt_draw.h**: Calls `BuildTables()` (defined in rt_draw.c, uses globals from rt_view.h like `scale`, `heightnumerator`, `pixelangle[]`)
- **rt_main.c**: Likely calls `SetupScreen()`, `SetViewSize()`, `LoadColorMap()` during engine initialization and mode switching
- **rt_game.c**: Calls `UpdateLightLevel()` per frame; reads `fulllight` global during game logic
- **rt_playr.c**: May call `UpdateLightning()` when lightning effects trigger
- **rt_menu.h / Control Panel**: Calls `SetViewSize()` when player adjusts view size in options
- **rt_net.c / Networking**: Calls `SetModemLightLevel()` for modem/bandwidth-optimized rendering
- **Raycasting inner loop** (rt_draw.c): Reads `pixelangle[]` per pixel, `colormap`/`greenmap`/`redmap` per scanline, `normalshade`/`maxshade` per wall segment

### Outgoing (what this file depends on)
- **modexlib.h**: VGA ModeX video mode constants, screen buffer management primitives (MAXVIEWWIDTH implied to be 320 pixels)
- **rt_def.h** (implicit via modexlib.h): Base types (`byte`, `longword`, `fixed`, `boolean`)
- **Gamma/Palette Resources** (loaded at runtime): Gamma table data and colormap/palette data (no #include visible, likely linked or loaded from .RTC files)

## Design Patterns & Rationale

### Separation of Concerns via Globals
The file uses **explicit global state** rather than opaque handles. This is typical of 1990s DOS engines: each global directly feeds the rendering loop without indirection. `pixelangle[]` is a precomputed lookup table (computed once per resolution change) to avoid expensive angle calculations per ray.

### Tiered Lighting Control
Three levels of illumination control:
1. **Static** (`SetIllumination()`) – sets absolute brightness (used in menus, cinematics)
2. **Dynamic per-area** (`UpdateLightLevel(area)`) – reads map sector lighting (used in-game)
3. **Transient effects** (`UpdateLightning()`) – animates lightning flashes per frame

This mirrors Doom-era level design: sectors have base light levels, and effects modulate on top.

### Focal Width (FOV) as Separate Concept
`focalwidth`, `FOCALWIDTH`, `FPFOCALWIDTH` suggest FOV is decoupled from viewport size:
- **Viewport size** affects resolution (11 configurable sizes via `SetViewSize()`)
- **Focal width** affects apparent FOV without recomputing raycasting geometry

This allows independent tuning of performance (viewport size) and visual field (FOV).

### Gamma as Lookup Table (LUT)
`gammatable[GAMMAENTRIES]` with 8 levels × 64 entries suggests per-pixel gamma correction: each screen color is mapped through `gammatable[gammaindex * 64 + original_color]` during rasterization. Precomputing this avoids runtime exponentiation.

## Data Flow Through This File

```
Game Init / Mode Selection
  └─→ SetupScreen(flip) + LoadColorMap()
      └─→ Initialize: viewheight, viewwidth, pixelangle[], colormap, scale, heightnumerator, centerx, centery, screenofs
      
Per-Frame Raycasting Loop (rt_draw.c)
  ├─→ Read: pixelangle[x] → angle per pixel
  ├─→ Read: scale, heightnumerator → 3D→2D projection
  ├─→ Read: normalshade, maxshade, minshade → wall fill brightness
  └─→ Read: colormap[shade * 256 + color] → shaded color output

Game Events (e.g., player enters dark room)
  └─→ UpdateLightLevel(area) → modifies: fulllight, normalshade, maxshade, minshade
      └─→ Read by next raycasting frame

Lightning Flash (e.g., enemy attacks)
  └─→ UpdateLightning() → modifies: lightning flag, lightninglevel
      └─→ Applied to shading tables in next frame

Menu: Player Adjusts FOV
  └─→ ChangeFocalWidth(+5) → updates: focalwidth
      └─→ Affects raycaster projection next frame

Menu: Player Changes View Size (performance)
  └─→ SetViewSize(8) → recomputes: viewheight, viewwidth, scale, heightnumerator, pixelangle[]
      └─→ Full viewport reconfiguration for next raycasting loop
```

## Learning Notes

### Idiomatic to 1990s DOS/VGA Engine
- **Precomputed lookup tables** (`pixelangle[]`, `gammatable[]`, `mapmasks1/2/3[]`) avoid runtime math
- **Global state over handles**: No opaque viewport or lighting objects—globals are the API contract
- **Palette-based rendering**: 8-bit indexed color with `colormap[]` variants for shading (different palettes for different light levels)
- **Fixed-point math implied**: Types like `fixed` and `longword` for scale/projection without floating-point

### Raycasting-Specific Patterns
- **Angle-per-pixel lookup** (`pixelangle[]`) is core to raycasting: each column of screen maps to a ray direction
- **Shade tables** (`normalshade`, `maxshade`, `minshade`) are per-level configuration; darkness animates by adjusting these bounds
- **Colormap indexing** as `colormap[shade_level * 256 + base_color]` is fast palette lookup for 8-bit video

### Modern Contrast
Modern 3D engines (Unity, UE4) use:
- Floating-point camera matrices instead of fixed FOV constants
- GPU textures + shaders instead of palette lookup tables
- Dynamic lighting (per-pixel) instead of per-sector tables
- Gamma correction in sRGB space, not LUT-based

## Potential Issues

1. **Lighting Update Lag**: `UpdateLightLevel()` reads map area but doesn't specify how light interpolates between frames. If the transition is instant, players may see jarring brightness shifts; if too slow, dark areas feel unresponsive.

2. **Gamma Table Assumption**: `GAMMAENTRIES = 64*8` and 11 view sizes suggest the lookup is optimized for 8-bit indexed color. Modern displays with 32-bit True Color would need a different approach (or the LUT is upsampled at display time).

3. **Modem Light Optimization**: `SetModemLightLevel(type)` and `GetLightRateTile()` suggest special handling for modem games (bandwidth-limited), but it's unclear if this path coexists cleanly with local lighting or if it's a wholesale replacement.

4. **No Viewport Bounds Validation**: `SetViewSize()` takes an index but the header doesn't document valid range (0–10 for MAXVIEWSIZES=11, presumably). Invalid indices could cause buffer overruns if not checked in the .c implementation.
