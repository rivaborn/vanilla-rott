# rott/rt_view.h

## File Purpose
Header for the rendering view subsystem. Declares constants, global state, and functions for managing screen setup, camera/focal width parameters, color palettes, gamma correction, and dynamic lighting/illumination levels in a 1990s-era software 3D engine.

## Core Responsibilities
- Screen initialization and view size configuration (MAXVIEWSIZES: 11 configurable sizes)
- Focal width (field-of-view) adjustment
- Gamma table management (8 levels, 64×8 entries)
- Player color palette selection (11 colors) and colormap loading
- Lighting/illumination system: dynamic per-area levels, darkness/shade ranges, and lightning effects
- Status bar display control (kills, health stats, bottom/top bars)
- Per-tile light query functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `playercolors` | enum | Enumeration of 11 player color options (gray, brown, black, tan, red, olive, blue, white, green, purple, orange) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `StatusBar` | int | extern | Bitmask controlling visibility of kills counter, health bar, bottom/top status bars |
| `playermaps[MAXPLAYERCOLORS]` | byte* array | extern | Color palette pointers for each player color |
| `pixelangle[MAXVIEWWIDTH]` | short array | extern | Lookup table mapping screen pixels to angles (raycasting) |
| `gammatable[GAMMAENTRIES]` | byte array | extern | Gamma correction lookup table (8 levels × 64 entries) |
| `gammaindex` | int | extern | Current gamma correction level |
| `uniformcolors[MAXPLAYERCOLORS]` | byte array | extern | Uniform colors per player palette |
| `mapmasks1/2/3[4][9]` | byte arrays | extern | Post-scaling map masks (3 tables, 4×9 each) |
| `normalshade`, `maxshade`, `minshade` | int | extern | Shading level bounds (base and current) |
| `baseminshade`, `basemaxshade` | int | extern | Base shading level bounds |
| `viewheight`, `viewwidth` | int | extern | Current viewport dimensions |
| `heightnumerator`, `scale` | longword / fixed | extern | Scaling factors for height and general rendering |
| `screenofs`, `centerx`, `centery`, `centeryfrac` | int | extern | Screen buffer offset and viewport center coordinates |
| `fulllight` | int | extern | Full brightness reference level |
| `colormap`, `greenmap`, `redmap` | byte* | extern | Colormap variants for shading |
| `weaponscale`, `viewsize` | int | extern | Weapon sprite scale and view size index |
| `focalwidth` | int | extern | Focal width (FOV-related) |
| `yzangleconverter` | int | extern | Angle conversion factor for Y-Z plane |
| `lightninglevel` | int | extern | Current lightning intensity |
| `lightning` | boolean | extern | Lightning effect active flag |
| `darknesslevel` | int | extern | Current darkness/shade level |

## Key Functions / Methods

### SetupScreen
- **Signature:** `void SetupScreen(boolean flip)`
- **Purpose:** Initialize or reinitialize the screen/viewport configuration.
- **Inputs:** `flip` – boolean flag (likely for page-flipping in double-buffering)
- **Outputs/Return:** None
- **Side effects:** Modifies global view configuration (viewheight, viewwidth, screenofs, etc.)
- **Calls:** Not inferable from this file
- **Notes:** Core initialization entry point for view system

### SetViewSize
- **Signature:** `void SetViewSize(int size)`
- **Purpose:** Change viewport size (one of 11 pre-defined sizes).
- **Inputs:** `size` – index into view size configuration (0–10)
- **Outputs/Return:** None
- **Side effects:** Updates viewheight, viewwidth, scale, heightnumerator globals
- **Calls:** Not inferable from this file
- **Notes:** Likely recalculates rendering scale factors

### ResetFocalWidth / ChangeFocalWidth
- **Signature:** `void ResetFocalWidth(void)` / `void ChangeFocalWidth(int amount)`
- **Purpose:** Manage focal width (field-of-view); reset to default or adjust by amount.
- **Inputs:** `amount` – signed delta to focal width
- **Outputs/Return:** None
- **Side effects:** Updates `focalwidth` global
- **Calls:** Not inferable from this file
- **Notes:** Separate functions for reset vs. delta adjustment

### LoadColorMap
- **Signature:** `void LoadColorMap(void)`
- **Purpose:** Load/initialize color palette from resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes `colormap`, `greenmap`, `redmap` and player color palettes
- **Calls:** Not inferable from this file

### UpdateLightLevel / SetIllumination
- **Signature:** `void UpdateLightLevel(int area)` / `void SetIllumination(int level)`
- **Purpose:** Update lighting for a map area or set absolute illumination level.
- **Inputs:** `area` – map sector/area ID; `level` – illumination value
- **Outputs/Return:** None
- **Side effects:** Updates `fulllight`, `lightninglevel`, shading globals
- **Calls:** Not inferable from this file
- **Notes:** `UpdateLightLevel` is dynamic (per-area), `SetIllumination` is static (absolute)

### GetIlluminationDelta
- **Signature:** `int GetIlluminationDelta(void)`
- **Purpose:** Query the change in illumination (rate of change).
- **Inputs:** None
- **Outputs/Return:** Integer delta value
- **Side effects:** None (query function)
- **Calls:** Not inferable from this file

### UpdateLightning
- **Signature:** `void UpdateLightning(void)`
- **Purpose:** Process lightning effect animation/state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies `lightning`, `lightninglevel` globals
- **Calls:** Not inferable from this file
- **Notes:** Called per frame

### SetLightLevels / SetupLightLevels
- **Signature:** `void SetLightLevels(int darkness)` / `void SetupLightLevels(void)`
- **Purpose:** Configure shading bounds based on darkness level or initialize defaults.
- **Inputs:** `darkness` – darkness/shade intensity
- **Outputs/Return:** None
- **Side effects:** Updates `normalshade`, `maxshade`, `minshade` globals
- **Calls:** Not inferable from this file
- **Notes:** `SetupLightLevels` is initialization; `SetLightLevels` is per-frame update

### SetLightRate / GetLightRate
- **Signature:** `void SetLightRate(int rate)` / `int GetLightRate(void)`
- **Purpose:** Set or query the light transition speed.
- **Inputs:** `rate` – transition rate
- **Outputs/Return:** Integer rate value
- **Side effects:** None (getter); setter affects light update rate
- **Calls:** Not inferable from this file

### SetModemLightLevel / GetLightRateTile / GetLightLevelTile
- **Signature:** `void SetModemLightLevel(int type)` / `int GetLightRateTile(void)` / `int GetLightLevelTile(void)`
- **Purpose:** Modem-mode-specific light configuration and per-tile light queries.
- **Inputs:** `type` – modem light mode
- **Outputs/Return:** Integer (light level or rate per tile)
- **Side effects:** Modem setter affects light mode; getters query tile-specific values
- **Calls:** Not inferable from this file
- **Notes:** Specialized for multiplayer/modem gameplay

### DrawCPUJape
- **Signature:** `void DrawCPUJape(void)`
- **Purpose:** Render CPU performance message (defined at Y=YOURCPUSUCKS_Y, height=YOURCPUSUCKS_HEIGHT).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Screen drawing
- **Calls:** Not inferable from this file
- **Notes:** Debug/HUD element for CPU load indication

## Control Flow Notes
This is a view/rendering subsystem header. Entry points are:
- **Init:** `SetupScreen(flip)`, `LoadColorMap()`, `SetupLightLevels()`
- **Per-frame:** `UpdateLightLevel(area)`, `UpdateLightning()`, `SetLightLevels(darkness)`, raycasting setup via pixelangle lookup
- **Config:** `SetViewSize(size)`, `ChangeFocalWidth(amount)`
- **Query:** Illumination/light rate getters for game logic

## External Dependencies
- **Includes:** `modexlib.h` (VGA ModeX video mode constants and screen buffer management)
- **Implied definitions:** `rt_def.h` (via modexlib), defining types like `byte`, `longword`, `fixed`, `boolean`
- **Defined elsewhere:** All function implementations; colormap resources; gamma/palette data
