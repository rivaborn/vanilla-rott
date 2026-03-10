# rott/rt_draw.c

## File Purpose

Core 3D rendering engine implementing raycasting walls and scaled sprite drawing. Manages coordinate transformations, lighting, double-buffering, and special visual effects for a software-rendered first-person view.

## Core Responsibilities

- **Coordinate transformation**: Convert world 3D coordinates to 2D screen space via perspective projection
- **Wall rendering**: Raycasting-based column-by-column wall drawing with texture interpolation
- **Sprite rendering**: Draw scaled actors and static objects with depth sorting
- **Lighting/shading**: Apply distance-based fog, light sources, and fullbright effects to surfaces
- **Double-buffering & page flipping**: Manage video memory swapping with screen shake effects
- **Weapon/UI rendering**: Draw first-person weapon and status elements
- **Special effects**: Rotation, zoom, screen transitions, fades, title sequences
- **Door/pushwall rendering**: Interpolate multi-level masked wall textures

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `visobj_t` | struct | Visible object queued for rendering (shape, position, height, lighting) |
| `wallcast_t` | struct | Wall post data from raycasting (texture, height, lighting, clipping) |
| `patch_t` | struct | Sprite/texture header with size and column offsets |
| `transpatch_t` | struct | Transparent sprite variant with translucency data |
| `screensaver_t` | struct | Screen saver rotation animation state |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `vislist` | `visobj_t[]` | global | Queue of visible objects awaiting rendering each frame |
| `visptr` | `visobj_t*` | global | Current write pointer into vislist |
| `tilemap` | `word[][]` | global | Map wall tile values for collision/visibility checks |
| `spotvis` | `byte[][]` | global | Visibility bitmap per map tile |
| `mapseen` | `byte[][]` | global | Explored area tracking for map display |
| `shadingtable` | `byte*` | global | Current palette remap for lighting effects |
| `sintable`, `costable`, `tantable` | fixed[] | global | Precomputed trig lookup tables for fast math |
| `lights` | `unsigned long*` | global | Light level map for dynamic lighting calculations |
| `viewx`, `viewy`, `viewangle` | fixed, int | global | Camera position and orientation |
| `pheight`, `nonbobpheight` | int | global | Camera height with/without weapon bob |
| `weaponbobx`, `weaponboby` | int | static | Weapon sway animation offsets |
| `sortedvislist` | `visobj_t*[]` | static | Sorted array of visible objects for depth-correct rendering |
| `pretics[]` | int[] | static | Adaptive detail frame time tracking |
| `RotatedImage` | `byte*` | static | Off-screen buffer for rotation effects |
| `ScreenSaver` | `screensaver_t*` | static | Screen saver animation state |

## Key Functions / Methods

### ThreeDRefresh
- **Signature**: `void ThreeDRefresh(void)`
- **Purpose**: Main per-frame rendering entry point; orchestrates entire 3D refresh cycle
- **Inputs**: None (reads global `viewx`, `viewy`, `player`, game state)
- **Outputs/Return**: None (writes to video buffer)
- **Side effects**: Clears buffer, renders walls/sprites/weapon/HUD, flips screen, updates gameframe counter
- **Calls**: `RefreshClear`, `WallRefresh`, `DrawPlanes`, `DrawScaleds`, `DrawPlayerWeapon`, `DrawStats`, `FlipPage`, `DrawMessages`, `DrawPlayerLocation`
- **Notes**: Handles multiple view modes (normal, missile, development player cycling); guards missile/god mode HUD elements

### WallRefresh
- **Signature**: `void WallRefresh(void)`
- **Purpose**: Perform raycasting and populate wall post array; apply camera effects (height, bobbing, view angle)
- **Inputs**: Global camera state, map visibility, missile/player objects
- **Outputs/Return**: None (populates `posts[]` array via `Refresh()`)
- **Side effects**: Sets view matrices, applies weapon bob, triggers raycasting refresh
- **Calls**: `Refresh`, `TransformPushWalls`, `TransformDoors`, `DrawWalls`
- **Notes**: Handles psychedelic shroom mode via sinusoidal view angle/focal width distortion; sets up yz-angle camera tilt

### DrawScaleds
- **Signature**: `void DrawScaleds(void)`
- **Purpose**: Collect, sort, and draw all scaled visible objects (actors, items, shadows)
- **Inputs**: Global actor/static lists, visibility arrays
- **Outputs/Return**: None
- **Side effects**: Populates `vislist` with transformed actor data, sorts by depth, calls rendering functions
- **Calls**: `TransformObject`, `StatRotate`, `CalcRotate`, `SetSpriteLightLevel`, `SetColorLightLevel`, `SortVisibleList`, `ScaleShape`, `ScaleTransparentShape`, `ScaleSolidShape`, `InterpolateDoor`, `InterpolateMaskedWall`
- **Notes**: Handles rotation/direction calculation, height-flipping for disks/pillars, translucency/colored overlays; checks 3×3 tile visibility radius

### TransformObject
- **Signature**: `boolean TransformObject(int x, int y, int *dispx, int *dispheight)`
- **Purpose**: Project 3D world point to 2D screen with perspective falloff
- **Inputs**: World coordinates `(x, y)`; outputs screen x and height
- **Outputs/Return**: `true` if visible (z > MINZ), `false` if behind camera
- **Side effects**: None
- **Calls**: `FixedMul` (fixed-point multiply)
- **Notes**: Uses view-relative rotation matrix and perspective divide; typical raycaster approach

### TransformPoint
- **Signature**: `void TransformPoint(int x, int y, int *screenx, int *height, int *texture, int vertical)`
- **Purpose**: Project 3D point to screen with edge clipping and texture coordinate wrapping
- **Inputs**: World `(x, y)`, vertical orientation flag
- **Outputs/Return**: Screen position, height, texture offset (wrapped to 16-bit)
- **Side effects**: None
- **Calls**: `FixedScale`, `FixedMul`
- **Notes**: Handles screen boundary clipping by computing intersection with FOV frustum edges; used for wall endpoint projection

### TransformPlane
- **Signature**: `boolean TransformPlane(int x1, int y1, int x2, int y2, visobj_t *plane)`
- **Purpose**: Project wall/door edge pair to screen space; validate visibility
- **Inputs**: Two world points, output visobj structure
- **Outputs/Return**: `true` if valid visible, `false` if off-screen
- **Side effects**: Populates plane with transformed coordinates and texture bounds
- **Calls**: `TransformSimplePoint`, `TransformPoint`
- **Notes**: Handles partial visibility (clipping); recomputes texture coords for screen-edge-clipped endpoints

### DrawWallPost
- **Signature**: `void DrawWallPost(wallcast_t *post, byte *buf)`
- **Purpose**: Render single raycasted wall column (with optional bottom section for doors)
- **Inputs**: Wall post data, framebuffer pointer
- **Outputs/Return**: None
- **Side effects**: Writes pixels to framebuffer; sets post ceiling/floor clipping
- **Calls**: `FixedMul`, `R_DrawWallColumn`
- **Notes**: Handles alt-tile floors (e.g., lower wall in multi-height rooms); applies clip bounds

### DrawWalls
- **Signature**: `void DrawWalls(void)`
- **Purpose**: Iterate raycasted posts and draw columns; handles detail level (doublestep)
- **Inputs**: Global `posts[]` array from raycasting, `doublestep` detail level
- **Outputs/Return**: None
- **Side effects**: Renders wall columns to framebuffer via `R_DrawWallColumn`
- **Calls**: `SetWallLightLevel`, `DrawWallPost`, `VGAMAPMASK`, `VGAWRITEMAP`
- **Notes**: Adaptive detail: at high doublestep, renders 2 columns per pixel plane; manages VGA planar writes

### DrawPlayerWeapon
- **Signature**: `void DrawPlayerWeapon(void)`
- **Purpose**: Draw first-person weapon sprite with animation and positioning offsets
- **Inputs**: Player state (weapon, frame, height, uniform color), weapon scale
- **Outputs/Return**: None
- **Side effects**: Renders 1–2 weapon sprites (dual pistols)
- **Calls**: `ScaleWeapon`
- **Notes**: Handles weapon-specific Y offsets; scales by `weaponscale` with bob modulation; shareware/registered variant handling

### SortVisibleList
- **Signature**: `void SortVisibleList(int numvisible, visobj_t *vlist)`
- **Purpose**: Depth-sort visible objects for painter's algorithm (back to front)
- **Inputs**: Array of visible objects
- **Outputs/Return**: None
- **Side effects**: Populates `sortedvislist` in sorted order
- **Calls**: `hsort` with custom comparator
- **Notes**: Uses `CompareHeights` comparator on `viewheight` field; heap sort for O(n log n)

### SetSpriteLightLevel
- **Signature**: `void SetSpriteLightLevel(int x, int y, visobj_t *sprite, int dir, int fullbright)`
- **Purpose**: Compute and apply lighting to sprite based on distance fog and light sources
- **Inputs**: World position, sprite object, direction (for light source sampling), fullbright flag
- **Outputs/Return**: Sets `sprite->colormap`
- **Side effects**: None
- **Calls**: `LightSourceAt`
- **Notes**: Handles gas/fulllight/fog modes; light source lookup uses directional bias (east/west vs. north/south)

### SetColorLightLevel
- **Signature**: `void SetColorLightLevel(int x, int y, visobj_t *sprite, int dir, int color, int fullbright)`
- **Purpose**: Apply color-indexed lighting to player/colored sprites
- **Inputs**: World position, sprite, color index, fullbright flag
- **Outputs/Return**: Sets `sprite->colormap` to player color palette + lighting offset
- **Side effects**: None
- **Calls**: `LightSourceAt`
- **Notes**: Similar to SetSpriteLightLevel but uses colored palette maps

### CalcHeight
- **Signature**: `int CalcHeight(void)`
- **Purpose**: Compute wall height from precomputed ray intercept
- **Inputs**: Global `xintercept`, `yintercept`, `viewx`, `viewy`
- **Outputs/Return**: Perspective-scaled height
- **Side effects**: None
- **Calls**: `FixedMul`
- **Notes**: Used by raycaster; simple fixed-point divide

### CalcRotate
- **Signature**: `int CalcRotate(objtype *ob)`
- **Purpose**: Determine sprite rotation frame based on camera angle relative to object
- **Inputs**: Object position and state
- **Outputs/Return**: Rotation index (0–15 or 0–7 depending on state)
- **Side effects**: None
- **Calls**: `atan2_appx`
- **Notes**: Handles 8 and 16-direction sprites; pain frame special case (4-way rotation)

### TransformDoors
- **Signature**: `void TransformDoors(void)`
- **Purpose**: Collect and sort visible doors for rendering
- **Inputs**: Door list from map
- **Outputs/Return**: None
- **Side effects**: Populates local door visibility list, calls `InterpolateWall` for rendering
- **Calls**: `TransformPlane`, `SortVisibleList`, `InterpolateWall`
- **Notes**: Similar to sprite rendering but for door geometry

### InterpolateWall
- **Signature**: `void InterpolateWall(visobj_t *plane)`
- **Purpose**: Texture-map a visible wall/door plane across screen columns
- **Inputs**: Transformed plane with texture bounds, screen coordinates
- **Outputs/Return**: None
- **Side effects**: Updates `posts[]` array with texture references
- **Calls**: `FixedMulShift`
- **Notes**: Linear interpolation of texture U coordinate; updates wall height if closer than existing post

### InterpolateDoor
- **Signature**: `void InterpolateDoor(visobj_t *plane)`
- **Purpose**: Render door sprites with per-plane rendering (4 VGA planes)
- **Inputs**: Door visobj with shape lumps
- **Outputs/Return**: None
- **Side effects**: Renders scaled door post columns to framebuffer
- **Calls**: `W_CacheLumpNum`, `FixedMul`, `SetLightLevel`, `ScaleMaskedPost`, `R_DrawWallColumn`, `VGAWRITEMAP`, `VGAREADMAP`
- **Notes**: Handles multi-level doors via `levelheight`; interleaves rendering across 4 VGA planes

### InterpolateMaskedWall
- **Signature**: `void InterpolateMaskedWall(visobj_t *plane)`
- **Purpose**: Render 3-layer masked wall (bottom, middle, top textures)
- **Inputs**: Masked wall visobj with 3 texture lumps
- **Outputs/Return**: None
- **Side effects**: Renders scaled posts to framebuffer
- **Calls**: `W_CacheLumpNum`, `ScaleTransparentPost`, `ScaleMaskedPost`, `VGAWRITEMAP`, `VGAREADMAP`
- **Notes**: Handles arbitrary `levelheight` wall stacking; uses transparency and masking

### FlipPage
- **Signature**: `void FlipPage(void)`
- **Purpose**: Swap framebuffer pointers and set VGA start address; apply screen shake
- **Inputs**: Global `bufferofs`, screen shake state
- **Outputs/Return**: None
- **Side effects**: Updates video DAC, cycles `bufferofs` through video memory pages
- **Calls**: `ScreenShake`, `OUTP` (hardware I/O)
- **Notes**: Triple-buffering by cycling through page1–page3; screen shake modulates start address per frame

### RotateBuffer
- **Signature**: `void RotateBuffer(int startangle, int endangle, int startscale, int endscale, int time)`
- **Purpose**: Animate 3D rotation and zoom of framebuffer over time
- **Inputs**: Start/end angles and scales, animation duration
- **Outputs/Return**: None
- **Side effects**: Renders rotated/scaled screen each frame
- **Calls**: `StartupRotateBuffer`, `ScaleAndRotateBuffer`, `ShutdownRotateBuffer`, `DrawRotatedScreen`, `FlipPage`, `CalcTics`
- **Notes**: Used for title sequences and transitions; allocates temp buffer

### ApogeeTitle
- **Signature**: `void ApogeeTitle(void)`
- **Purpose**: Play Apogee Software intro cinematic with rotating logo
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Draws sprite, plays music, animates rotation effect
- **Calls**: `StartupRotateBuffer`, `DrawRotatedScreen`, `FlipPage`, `MU_StartSong`, `CalcTics`
- **Notes**: Hardcoded timing and animation parameters

### DoCreditScreen
- **Signature**: `void DoCreditScreen(void)`
- **Purpose**: Scroll game credits with animated text warp
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Draws credits to screen with fade/scroll effects
- **Calls**: `WarpCreditString`, `DrawBackground`, `DrawPreviousCredits`, `FlipPage`, `VL_FadeIn`, `VL_FadeOut`, `MenuFadeOut`
- **Notes**: Two separate credit sequences; uses sound effects on text appearance

### DrawRotatedScreen
- **Signature**: `void DrawRotatedScreen(int cx, int cy, byte *destscreen, int angle, int scale, int masked)`
- **Purpose**: Render off-screen buffer with 2D rotation and scaling applied
- **Inputs**: Center, angle, scale, screen buffer, mask flag
- **Outputs/Return**: None
- **Side effects**: Rasterizes rotated image to framebuffer
- **Calls**: `DrawRotRow`, `DrawMaskedRotRow`, `VGAWRITEMAP`, `FixedMulShift`
- **Notes**: Fixed-point 2D transform matrix; per-scanline inner loop for row drawing

### DoEndCinematic
- **Signature**: `void DoEndCinematic(void)` (two variants: shareware & registered)
- **Purpose**: Play end-game cinematic sequence with narration, music, and sprite animations
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Complex multi-part sequence with fades, sprite movements, text scrolling
- **Calls**: `ShowFinalDoor`, `ShowTransmitter`, `DoTransmitterExplosion`, `DoBurningCastle`, `DestroyEarth`, `PlayerQuestionScreen`, `DIPCredits`, various Draw/Warp functions
- **Notes**: Registered version only (shareware has simpler ending); uses extensive palette/music changes

### CalcTics
- **Signature**: `void CalcTics(void)`
- **Purpose**: Compute elapsed frame time in ticks; support profiling and adaptive detail
- **Inputs**: Global `ticcount`, `oldtime`
- **Outputs/Return**: Sets global `tics`
- **Side effects**: Updates `oldtime`; modulates `tics` for demo recording
- **Calls**: `ISR_SetTime`
- **Notes**: Adaptive timing for demo playback; visual tick meter in dev mode

### BuildTables
- **Signature**: `void BuildTables(void)`
- **Purpose**: Load and initialize math lookup tables (sin/cos, tangent) from WAD
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Populates `sintable`, `costable`, `tantable`, `angletodir`
- **Calls**: `W_CacheLumpName`, `W_GetNumForName`, `CheckVendor`
- **Notes**: Foundation for all fixed-point math; critical for engine startup

---

## Control Flow Notes

**Initialization**: `BuildTables()` loads lookup tables once at startup.

**Per-Frame Render Loop** (from `ThreeDRefresh()`):
1. Clear buffer via `RefreshClear()`
2. Call `WallRefresh()` to raycasts walls, applies camera transforms/effects
3. Optional `DrawPlanes()` for ceiling/floor (if enabled)
4. Call `DrawScaleds()` to collect, sort, and draw sprites
5. Call `DrawPlayerWeapon()` to render HUD weapon
6. Conditionally draw status, messages, pause overlay
7. Call `FlipPage()` to swap buffers and update display

**Lighting Pipeline**:
- Wall lighting: `WallRefresh()` → `SetWallLightLevel()` per post → `DrawWallPost()` writes colored pixels
- Sprite lighting: `DrawScaleds()` → `SetSpriteLightLevel()` or `SetColorLightLevel()` → rendering functions apply colormap

**Visibility & Depth Sorting**:
- Raycaster generates wall heights per column
- Sprites collected in `vislist`, sorted by `viewheight` (painter's algorithm)
- Walls drawn first, then sprites from back to front via `sortedvislist`

**Special Effects**:
- Screen shake: `FlipPage()` modulates start address via `ScreenShake()`
- Rotation/zoom: Title sequences use `RotateBuffer()` → `DrawRotatedScreen()`
- Cinematic sequences hook into main loop via functions like `DoEndCinematic()`

## External Dependencies

- **Math**: `FixedMul`, `FixedDiv2`, `FixedScale`, `FixedMulShift` (watcom.h pragmas)
- **Raycasting core**: `Refresh()`, `R_DrawWallColumn()`, `R_DrawColumn()` (defined elsewhere, likely rt_fc_a.h / assembly)
- **Sprite rendering**: `ScaleShape`, `ScaleTransparentShape`, `ScaleSolidShape`, `ScaleMaskedPost`, `ScaleWeapon` (rt_scale.h or rt_dr_a.h)
- **Resource management**: `W_CacheLumpName`, `W_GetNumForName`, `W_CacheLumpNum` (w_wad.h)
- **Memory**: `SafeMalloc`, `SafeFree` (z_zone.h)
- **Video**: `VGAWRITEMAP`, `VGAREADMAP`, `VGAMAPMASK`, `OUTP` (modexlib.h, VGA hardware control)
- **Camera/actors**: `player`, `firstactive`, `firstactivestat`, `objtype`, `statobj_t` (rt_actor.h)
- **Map data**: `tilemap`, `spotvis`, `doorobjlist`, `doornum`, `pwallobjlist`, `pwallnum` (rt_main.h likely)
- **Sound/music**: `MU_StartSong`, `SD_Play` (rt_sound.h)
- **UI**: `DrawStats`, `DrawMessages`, `DrawScreenSprite`, `DrawNormalSprite` (other modules)
- **Math tables**: `sintable`, `costable`, `tantable`, `gammatable`, `colormap`, `playermaps`, `redmap` (global)
- **Lighting**: `LightSourceAt`, `lights` array (rt_floor.h or similar)
- **Input**: `UpdateClientControls` (modem/netgame support, rt_net.h)

**Assembler/Hardware I/O**:
- Direct VGA mode-X register writes via `OUTP()`
- Likely inline assembly for `FixedMul` and related operations (Watcom pragmas)
- ISR callbacks for timer (`ISR_SetTime`, from isr.h)
