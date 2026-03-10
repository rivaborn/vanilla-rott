# rott/rt_view.c

## File Purpose
Manages the 3D view rendering pipeline, including projection calculations, lighting systems, color mapping, and screen layout. Handles focal width adjustments, lightning/periodic lighting effects, and illumination management for the ROTT engine's raycasting renderer.

## Core Responsibilities
- View geometry: projection angles, focal width, screen dimensions, scaling factors
- Lighting: shade levels, dynamic area lighting, lightning flashes, periodic light oscillations
- Color management: loading and applying colormaps for player colors and lighting effects
- Screen setup: UI layout (status bars, kills display), viewport positioning and scaling
- Illumination control: temporary light level adjustments for special effects

## Key Types / Data Structures
None defined in this file. Uses primitives and structures from included headers (pic_t from lumpy.h).

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|--------------------------------|---------|
| StatusBar | int | global | Bitfield controlling which UI bars are displayed |
| viewwidth, viewheight | int | global | Viewport dimensions in pixels |
| centerx, centery, centeryfrac | int | global | Viewport center coordinates (fixed-point) |
| scale | fixed | global | Scaling factor for projection calculations |
| heightnumerator | longword | global | Numerator for height calculations in raycasting |
| focalwidth | int | global | Current focal length; adjustable via changers |
| screenofs | int | global | Byte offset into video buffer for viewport |
| weaponscale | int | global | Weapon sprite scaling based on viewheight |
| colormap, redmap, greenmap | byte* | global | Lighting/effect color lookup tables (256 bytes each) |
| playermaps[11] | byte* array | global | Player-color-specific lighting tables |
| pixelangle[320] | short array | global | Angle for each screen pixel column |
| lightninglevel, lightning | int/bool | global | Current lightning brightness and enabled flag |
| minshade, maxshade, baseminshade, basemaxshade | int | global | Current and base shade (brightness) limits |
| normalshade, darknesslevel | int | global | Light rate and darkness index |
| YourComputerSucksString | char* | static | Debug message ("Buy a 486! :)") |
| viewsizes[22] | int array | static | Preset (width,height) pairs for 11 view sizes |
| ColorMapLoaded | int | static | Guard to prevent LoadColorMap reentry |
| lightningtime, lightningdelta, lightningdistance, lightningsoundtime | int | static | Lightning animation state |
| periodic, periodictime | bool/int | static | Periodic lighting oscillation state |

## Key Functions / Methods

### ResetFocalWidth
- Signature: `void ResetFocalWidth(void)`
- Purpose: Reset focal width to default constant and recalculate projection scalars
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies global `focalwidth` to FOCALWIDTH (160); calls SetViewDelta to update `scale` and `heightnumerator`
- Calls: SetViewDelta
- Notes: Undoes any focal width changes; always called by SetViewSize to ensure consistency

### ChangeFocalWidth
- Signature: `void ChangeFocalWidth(int amount)`
- Purpose: Adjust focal width by a relative amount for zoom effects
- Inputs: amount (pixel offset relative to FOCALWIDTH)
- Outputs/Return: None
- Side effects: Sets `focalwidth = FOCALWIDTH + amount`; triggers SetViewDelta
- Calls: SetViewDelta
- Notes: Used by cinematic or zoom mechanics

### SetViewDelta
- Signature: `void SetViewDelta(void)`
- Purpose: Calculate projection constants from current focal width and center point
- Inputs: None (reads globals `centerx`, `focalwidth`)
- Outputs/Return: None (writes globals `scale`, `heightnumerator`)
- Side effects: Recomputes raycasting projection math
- Calls: None
- Notes: Internal helper; must be called whenever focalwidth or centerx changes

### CalcProjection
- Signature: `void CalcProjection(void)`
- Purpose: Load angle lookup table from WAD and compute per-pixel ray angles for all screen columns
- Inputs: None (reads global `centerx`)
- Outputs/Return: None (writes global `pixelangle[centerx*2]`)
- Side effects: Loads "tables" lump from WAD; allocates/frees temporary buffer; symmetric left/right angle setup
- Calls: W_CacheLumpName, memcpy, SafeMalloc, SafeFree
- Notes: Angle table scaled by 65536 internally; result stored as signed shorts; centermost pixels bisect angle values

### SetViewSize
- Signature: `void SetViewSize(int size)`
- Purpose: Configure entire view pipeline: dimensions, centering, status bars, scaling, and projection angles
- Inputs: size (0–10 index into viewsizes preset table)
- Outputs/Return: None (sets 15+ globals)
- Side effects: Modifies viewwidth, viewheight, screenofs, centerx, centery, centeryfrac, weaponscale, yzangleconverter, StatusBar; calls ResetFocalWidth, CalcProjection
- Calls: Error, SHOW_KILLS, ResetFocalWidth, CalcProjection
- Notes: Validates size in range [0,11); handles size-dependent UI (top bar at size<9, bottom bar at size<8, transparent health at size<10); clamps view height to 168 to prevent weapon scaling artifacts; centers view on 320×200 screen

### DrawCPUJape
- Signature: `void DrawCPUJape(void)`
- Purpose: Display "Buy a 486! :)" message as a joke/nag screen
- Inputs: None
- Outputs/Return: None
- Side effects: Renders text at (160, 100+24+2) using game font
- Calls: VW_MeasurePropString, DrawGameString
- Notes: Only triggered for smallest view size (size==0); intentional 1994 hardware plug

### SetupScreen
- Signature: `void SetupScreen(boolean flip)`
- Purpose: Initialize viewport for new level or game state: apply view size, draw background UI, optionally refresh display
- Inputs: flip (if true, calls ThreeDRefresh and updates hidden buffer)
- Outputs/Return: None
- Side effects: Calls SetViewSize; loads "backtile" lump if size<7; calls DrawPlayScreen, ThreeDRefresh, VL_CopyDisplayToHidden conditionally
- Calls: SetViewSize, W_CacheLumpName, DrawTiledRegion, DrawCPUJape, DrawPlayScreen, ThreeDRefresh, VL_CopyDisplayToHidden
- Notes: Typical startup sequence for game frame

### LoadColorMap
- Signature: `void LoadColorMap(void)`
- Purpose: Load lighting and player color lookup tables from WAD lumps into aligned memory
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates ~130 KB (256×16 colors, aligned); loads lumps "colormap", "specmaps", "playmaps"; modifies colormap, redmap, greenmap, playermaps globals; corrects fire-color palette entries; sets ColorMapLoaded guard flag
- Calls: Error, W_GetNumForName, W_LumpLength, SafeMalloc, W_ReadLump
- Notes: 256-byte alignment ensures cache-efficient lookups; fire-color fix loops indices 16–31 mapping to 16–23 range; errors if called twice; greenmap is pointer alias into redmap (redmap+4096 bytes)

### SetupLightLevels
- Signature: `void SetupLightLevels(void)`
- Purpose: Read map-embedded lighting configuration and apply light level, rate, fog, and light-sourcing setup
- Inputs: None (reads MAPSPOT tile data at hardcoded coordinates)
- Outputs/Return: None (calls SetLightLevels, SetLightRate; may allocate lights array via Z_Malloc)
- Side effects: Modifies minshade, maxshade, fog, lightsource, periodic, lightning, lightninglevel; initializes lights array if light-sourcing enabled
- Calls: MAPSPOT, Error, Z_Malloc, memset, SetLightLevels, SetLightRate
- Notes: Map must place fog icon (104–105) at (2,0,1), light-source/lightsource icon (139) at (3,0,1), darkness level (216–223) at (2,0,0), light rate (252–267) at (3,0,0); errors on invalid combinations (fog+light-source disallowed); defaults light rate to 4 if missing tile

### SetLightLevels
- Signature: `void SetLightLevels(int darkness)`
- Purpose: Convert darkness parameter into min/max shade bounds, with separate formulas for fog vs normal mode
- Inputs: darkness (0–7 range extracted from map tile index)
- Outputs/Return: None (sets minshade, maxshade, baseminshade, basemaxshade, darknesslevel)
- Side effects: Global lighting state update
- Calls: None
- Notes: Non-fog: `baseminshade = 0x10 + (7-darkness)>>1`, `basemaxshade = 0x1f - darkness>>1`; fog: `baseminshade = darkness`, `basemaxshade = 0x10`; minshade/maxshade are runtime adjustable copies

### GetLightLevelTile
- Signature: `int GetLightLevelTile(void)`
- Purpose: Reverse-compute the map tile index that would correspond to current light levels
- Inputs: None (reads baseminshade, basemaxshade, fog)
- Outputs/Return: Tile index 216–223 range (LIGHTLEVELBASE + computed value)
- Side effects: None
- Calls: None
- Notes: Used to export current lighting back to map editor

### SetLightRate
- Signature: `void SetLightRate(int rate)`
- Purpose: Set how rapidly shade changes with distance; clamp to engine-safe range
- Inputs: rate (0–15 range, tile index 252–267 range maps to 0–15)
- Outputs/Return: None (sets global normalshade)
- Side effects: `normalshade = (HEIGHTFRACTION+8) - rate` clamped to [3, 14]
- Calls: None
- Notes: Controls light falloff; clamps ensure raycaster doesn't underflow/overflow

### GetLightRate
- Signature: `int GetLightRate(void)`
- Purpose: Reverse-compute light rate from current normalshade
- Inputs: None (reads normalshade)
- Outputs/Return: Tile-independent rate value
- Side effects: None
- Calls: None

### GetLightRateTile
- Signature: `int GetLightRateTile(void)`
- Purpose: Return map tile index equivalent to current light rate
- Inputs: None (reads normalshade)
- Outputs/Return: Tile index in 252–267 range (LIGHTRATEBASE + rate)
- Side effects: None
- Calls: None

### UpdateLightLevel
- Signature: `void UpdateLightLevel(int area)`
- Purpose: Dynamically adjust shade bounds based on count of active light sources in an area; smooth transitions
- Inputs: area (area index 0–46)
- Outputs/Return: None (adjusts minshade, maxshade)
- Side effects: Increments or decrements minshade/maxshade by 1 unit per call toward computed target; reads external globals numareatiles[area], LightsInArea[area]
- Calls: None
- Notes: Skips if fog enabled; targets computed as `baseminshade + (GENERALNUMLIGHTS - numlights)` and `basemaxshade - numlights`; clamped to valid bounds; provides smooth visual fade rather than snappy changes

### SetIllumination
- Signature: `void SetIllumination(int level)`
- Purpose: Apply temporary global brightness offset (positive=bright, negative=dark) with clamping
- Inputs: level (signed offset to apply)
- Outputs/Return: None (adjusts maxshade, minshade)
- Side effects: `maxshade -= level`, `minshade -= level` clamped to [0x10, 31]
- Calls: None
- Notes: Disabled (returns immediately) if fog enabled; used by lightning, explosions; note sign convention (positive level reduces shade value = brightens)

### GetIlluminationDelta
- Signature: `int GetIlluminationDelta(void)`
- Purpose: Query current global illumination offset from baseline
- Inputs: None (reads maxshade, basemaxshade)
- Outputs/Return: Difference (can be positive or negative)
- Side effects: None
- Calls: None
- Notes: Returns 0 if fog enabled

### UpdateLightning
- Signature: `void UpdateLightning(void)`
- Purpose: Animate lightning flashes with brightness and delayed sound effects; delegates to periodic lighting if active
- Inputs: None (reads/modifies lightning state; reads GameRandomNumber)
- Outputs/Return: None (updates lightninglevel, lightningtime, lightningdelta, lightningsoundtime; calls SetIllumination, SD_Play3D, SD_PlayPitchedSound)
- Side effects: Complex state machine with three branches: (1) periodic mode → call UpdatePeriodicLighting; (2) fog or lightning disabled → skip; (3) lightning enabled → animate flashes with random intensity/distance and play sound after delay
- Calls: UpdatePeriodicLighting, GameRandomNumber, SD_Play3D, SD_PlayPitchedSound, SetIllumination
- Notes: lightningdistance (0–255) determines brightness and sound pitch; sound plays on first flash, visual illumination applied immediately if distance < 100; sound delay is `distance >> 1`; pitch is `-(distance << 2)` (negative = lower pitch); state resets when lightningtime expires

### UpdatePeriodicLighting
- Signature: `void UpdatePeriodicLighting(void)`
- Purpose: Apply continuous sinusoidal light oscillation for ambient mood effects
- Inputs: None (reads/modifies periodictime)
- Outputs/Return: None (updates basemaxshade, baseminshade)
- Side effects: Looks up sine value via `sintable[periodictime]`, multiplies by PERIODICMAG (6), adds to PERIODICBASE (0x0f); increments periodictime by PERIODICSTEP (20) with wraparound at FINEANGLES (2048)
- Calls: FixedMul
- Notes: Creates smooth breathing light effect; baseminshade is always `basemaxshade - (GENERALNUMLIGHTS + 1)` to maintain contrast

### SetModemLightLevel
- Signature: `void SetModemLightLevel(int type)`
- Purpose: Quick preset lighting configuration for multiplayer/modem games (overrides map defaults)
- Inputs: type (enum: bo_light_dark, bo_light_normal, bo_light_bright, bo_light_fog, bo_light_periodic, bo_light_lightning)
- Outputs/Return: None
- Side effects: Writes MAPSPOT values at hardcoded config points (2,0,0), (3,0,0), (2,0,1), (3,0,1); calls SetupLightLevels; modifies periodic, lightning flags
- Calls: SetupLightLevels
- Notes: bo_light_normal case is empty (no-op); lightning preset only activates if sky!=0; periodic preset overrides fog to 0 and enables lightsource; provides network-safe lighting without editing map files

## Control Flow Notes
**Initialization sequence:** LoadColorMap (once at startup) → SetupScreen/SetViewSize (per-level or viewport change) → SetupLightLevels (per-level).

**Per-frame updates:** UpdateLightLevel (called per visible area) → UpdateLightning (if lightning enabled, or delegates to UpdatePeriodicLighting).

**Configuration:** SetModemLightLevel can override map settings for multiplayer; SetLightLevels / SetLightRate provide granular control.

The file bridges map data (MAPSPOT) into engine globals and handles dynamic lighting updates for the raycaster's real-time rendering loop.

## External Dependencies
- **w_wad.h**: W_CacheLumpName, W_GetNumForName, W_LumpLength, W_ReadLump (WAD lump loading: "tables", "colormap", "specmaps", "playmaps", "backtile")
- **z_zone.h**: Z_Malloc (memory allocation for lights array)
- **rt_util.h**: SafeMalloc, SafeFree (safe memory wrappers); FixedMul (fixed-point multiply)
- **rt_game.h**: GameRandomNumber (random number generator); MAPSPOT (map tile access); lights, lightsource, fog, gamestate, numareatiles, LightsInArea (external globals for light sourcing and map state)
- **rt_sound.h**: SD_Play3D, SD_PlayPitchedSound, SD_LIGHTNINGSND (sound effects)
- **rt_draw.h / rt_vid.h**: DrawGameString, VW_MeasurePropString, DrawTiledRegion, DrawPlayScreen, ThreeDRefresh, VL_CopyDisplayToHidden, ylookup (rendering/display)
- **modexlib.h**: Mode X graphics primitives (implicitly used via drawing functions)
- **luminance tables**: sintable (sine lookup; defined elsewhere)
