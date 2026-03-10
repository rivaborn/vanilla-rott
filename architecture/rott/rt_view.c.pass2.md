# rott/rt_view.c — Enhanced Analysis

## Architectural Role
rt_view.c is the **view configuration and dynamic lighting manager** of the raycaster engine. It bridges map-embedded configuration (MAPSPOT tiles) into engine globals that drive frame-by-frame rendering. The file manages three critical subsystems: (1) **projection geometry** (focal width, screen scaling, angle lookup tables for raycasting), (2) **lighting state** (shade bounds, light falloff, colormap selection), and (3) **dynamic effects** (lightning flashes, periodic breathing light, area-based light sourcing). Every 3D frame depends on globals configured here.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_game.c / rt_main.c**: Call `SetupScreen()` and `SetViewSize()` during level initialization and viewport resize events
- **Rendering loop (rt_draw.c implied)**: Per-frame reads of globals `pixelangle[]`, `scale`, `heightnumerator`, `centerx`, `centery` for raycaster projection
- **rt_stat.c / area rendering**: Calls `UpdateLightLevel(area)` per visible area to smoothly adjust shade based on active light sources
- **rt_actor.c / rt_playr.c**: May trigger `SetIllumination()` for explosions/flashes and `UpdateLightning()` during gameplay
- **rt_menu.h**: Calls `SetViewSize()` when player adjusts viewport from menu
- **Multiplayer (rt_net.h implied)**: Calls `SetModemLightLevel()` to override map defaults for network games

### Outgoing (what this file depends on)
- **rt_game.h**: Reads `MAPSPOT()` (map tile config), external globals `lights`, `lightsource`, `fog`, `numareatiles`, `LightsInArea`, `gamestate`
- **w_wad.h**: Loads lumps: `"tables"` (angle LUT), `"colormap"` (light tables), `"specmaps"` (red/green maps), `"playmaps"` (player colors), `"backtile"`
- **rt_draw.h**: Calls rendering functions; reads `ylookup[]` (scanline offsets)
- **rt_sound.h**: Calls `SD_Play3D()`, `SD_PlayPitchedSound()` for lightning effects
- **lookups.h (implied)**: Reads `sintable[]` for periodic lighting sine wave

## Design Patterns & Rationale

**Configuration via MAPSPOT tiles:** Map designers place special tiles at hardcoded coordinates (fog at 2,0,1; light-source at 3,0,1) to configure lighting. Data-driven and editable in-engine, but fragile (misplaced tiles break initialization). Avoids separate config files.

**Dual-mode lighting (fog vs normal):** Two completely different shade formulas coexist. Fog mode is a late feature; branching logic persists throughout (SetIllumination, UpdateLightLevel skip if fog enabled). Reflects iterative development rather than refactored design.

**Smooth light transitions:** `UpdateLightLevel()` increments shade bounds by 1 per call instead of snapping, creating fade effects. Requires per-frame updates; visible smoothness over instantaneous response.

**Preset view sizes + baked UI:** 11 preset (width, height) pairs control viewport *and* which UI bars appear. `SetViewSize()` is monolithic, handling geometry + UI layout. Simple but inflexible.

**256-byte-aligned colormaps:** Manual cache-line alignment (`((int)colormap + 255)&~0xff`) for fast dynamic lookups `colormap[shade*256 + color]`. Performance optimization for Pentium-era hardware; modern allocators hide this.

## Data Flow Through This File

1. **Initialization** (once per game/level):
   - `LoadColorMap()` → loads WAD lumps → allocates/aligns RAM
   - `SetupLightLevels()` → reads MAPSPOT config → calls SetLightLevels/SetLightRate → sets minshade/maxshade baselines

2. **Per-frame setup**:
   - `SetViewSize(size)` → computes viewport geometry → calls CalcProjection → calls ResetFocalWidth
   - `CalcProjection()` → loads "tables" lump → distributes angles to `pixelangle[]` (symmetric left-right)

3. **Per-visible-area dynamic updates**:
   - `UpdateLightLevel(area)` → smoothly adjusts minshade/maxshade toward computed targets based on active lights
   - `UpdateLightning()` → animates flashes → calls `SetIllumination()` to brighten scene → plays sound with distance-based pitch

4. **Raycaster per-pixel**: Uses `pixelangle[x]`, `scale`, `heightnumerator`, colormap pointers to render with perspective and lighting.

## Learning Notes

**1. DDA-style raycasting (Wolfenstein 3D era):**
Precomputed `pixelangle[]` per screen column and `heightnumerator / distance` for height lookup reflect classic raycasting math, not modern matrix transforms. Hand-tuned fixed-point for DOS performance.

**2. Lighting as texture dimension:**
Colormap is a 2D LUT: `colormap[shade_level * 256 + palette_index]`. This **light texture** mechanism (no per-texel math) was cheap on 1990s hardware; modern engines use shaders.

**3. Global illumination as hack:**
Lightning/periodic lighting modify minshade/maxshade globally, brightening the entire scene uniformly. No shadowing or localized effects; simple but effective for atmosphere.

**4. Map data as configuration:**
Using tile positions for config (instead of .ini/lumps) is idiomatic to early game engines—data-driven and editable in-editor, but fragile and undiscoverable.

**5. Fixed focal width + zoom:**
ROTT locks focal width at 160 units (~90° FOV) and provides zoom via `ChangeFocalWidth()`, simplifying precomputation. Modern games expose dynamic FOV.

## Potential Issues

**1. MAPSPOT dependency is hard-fail:**
Missing required tiles (fog icon at 2,0,1, light rate at 3,0,0) trigger `Error()` (fatal crash). No graceful fallback; maps **must** be correctly configured. Tight coupling between map content and initialization.

**2. ColorMapLoaded guard insufficient on modern systems:**
`ColorMapLoaded` flag prevents `LoadColorMap()` reentry but assumes single-threaded execution. No protection against callbacks/hooks triggering during load. Safe on DOS; potential race condition on threaded systems.

**3. Fire-color fix hardcoded and undocumented:**
Indices 16–31 mapping to 16–23 in colormap is hardcoded with no explanation. If colormap layout changes, this breaks silently.

**4. Fog + light-source mutual exclusion only checked at runtime:**
No compile-time guarantee; runtime `Error()` if both are enabled. Early validation would be clearer.
