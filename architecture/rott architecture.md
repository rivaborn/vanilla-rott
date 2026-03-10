# Subsystem Overview: Ray-casting Rendering Engine

## Purpose
The ray-casting rendering engine converts 3D world geometry and entities into 2D screen images by casting rays from the player viewpoint, determining visible wall geometry, and compositing scaled sprites with dynamic lighting. It extends the original Wolfenstein 3D raycaster with masked walls, elevated platforms, sprite scaling, floor/ceiling rendering, and per-light shading—creating the visual core of Rise of the Triad's immersive first-person perspective.

## Key Files
| File | Role |
|------|------|
| engine.c/h | Ray-casting algorithm: grid traversal, wall intersection detection, texture coordinate resolution |
| rt_draw.c/h | Main rendering pipeline: camera transformation, visibility culling, sprite composition, frame management |
| rt_scale.c/h | Sprite-to-screen projection: scaling calculations, transparency/masking, lighting application |
| rt_floor.c/h | Horizontal span rasterization: floor, ceiling, parallax sky rendering with perspective mapping |
| rt_view.c/h | View parameters: focal width, lighting system, color palettes, illumination state |
| rt_vid.c/h | VGA framebuffer I/O: dirty-rect tracking, palette effects, double-buffering, primitive drawing |
| modexlib.c/h | VGA Mode-X hardware: video memory layout, plane selection, page-flipping, VBlank sync |
| rt_dr_a.asm/h | Assembly post-drawing: vertical column rendering with shading lookup |
| rt_sc_a.asm/h | Assembly scaled columns: masked sprite rendering with texture filtering |
| rt_fc_a.asm/h | Assembly floor/ceiling/sky: horizontal span rendering, rotation, masking |
| rt_eng.asm/h | Assembly ray-casting: grid traversal, opaqueness testing, visibility marking |
| rt_map.c/h | Interactive minimap: zoom-able level visualization, explored region tracking |

## Core Responsibilities
- **Ray-casting core**: Traverse tile grid from player position with fixed-point arithmetic; detect wall intersections; resolve texture offsets and heights; mark visible tiles
- **Wall rendering**: Determine columns visible from each screen pixel; handle door geometry, animated wall frames, and masked wall transparency
- **Sprite projection**: Transform 3D actor positions to screen space; calculate height-based scaling; depth-sort with wall occlusion testing
- **Sprite rasterization**: Draw scaled texture columns (posts) with per-pixel color translation via shading tables; support transparent and masked rendering modes
- **Lighting system**: Calculate distance-based fog levels; apply dynamic per-tile light sources with radius falloff; manage lightning flashes and periodic illumination pulses
- **Floor/ceiling rendering**: Calculate perspective-correct texture coordinates for horizontal spans; apply shading and animated textures
- **Sky rendering**: Parallax-scrolling background with configurable horizon height; render sky geometry atop floor/ceiling planes
- **VGA hardware management**: Configure planar graphics mode; manage multiple framebuffer pages; synchronize page flips with vertical blank; handle color palette I/O
- **Double-buffering optimization**: Track dirty screen rectangles; minimize copy bandwidth via block-based update system; chain updates for smooth animation
- **Visual effects**: Screen shake, palette fades, rotation sequences, title screens, cinematic integration

## Key Interfaces & Data Flow

**Exposes to game logic:**
- `ThreeDRefresh()` — Main frame rendering entry point; called once per game loop iteration
- `CalcTics()` — Frame timing calculation; returns elapsed tics since last call
- `posts[]` — Array of wall geometry (height, texture, shading) indexed by screen column; populated by raycasting
- `vislist[]` / `visobj_t` — Depth-sorted list of visible sprites for HUD/UI integration
- Camera state: `viewx`, `viewy`, `viewangle`, `viewsin`, `viewcos`, `viewwidth`, `viewheight`, `centery`
- Screen state: `bufferofs`, `displayofs`, `screenofs` (video memory pointers); `ylookup[]` (Y→offset LUT)
- Lighting state: `lights[]` array, `fulllight` flag, `fog` intensity, `lightninglevel`
- Rendering tables: `colormap[]`, `playermaps[]` (player colors), `shadingtable[]`, `redmap[]` (damage flash)
- Control functions: `FlipPage()`, `TurnShakeOff()`, `VL_FadeOut()`, `VL_FadeIn()`

**Consumes from other subsystems:**
- **Map geometry**: `tilemap[][]` (wall types), `mapplanes[][]` (floor/ceiling heights), `MAPSPOT()` macro for grid lookup
- **Interactive objects**: `doorobjlist[]`, `maskobjlist[]`, `pwallobjlist[]` (door/wall state with open/close frames)
- **Actors/sprites**: `firstactive` (actor linked list), `firstactivestat` (static object list), `objtype` instances with position/sprite/height
- **Sprite definitions**: `sprites[][]` grid (actor type → sprite frame lookup), sprite header data (width, height, column offsets)
- **Resources**: `W_CacheLumpNum()`, `W_GetNumForName()` (WAD lumps for textures, sprites, fonts from w_wad.c)
- **Memory**: `SafeMalloc()`, `SafeFree()` (zone allocator from z_zone.c)
- **Player state**: `player` global (position, angle, view state from rt_playr.c)
- **Door state**: `rt_door.c` functions and door object states for masked wall rendering
- **Math tables**: `sintable[]`, `costable[]`, `tantable[]` (trig lookup); `gammatable` (gamma correction)
- **Input**: Player movement/aim commands via `player.state` and local variables
- **Configuration**: `viewsize` (HUD size), detail levels, gamma setting, HUD visibility flags (rt_cfg.c)

## Runtime Role

**Startup (engine init phase):**
1. Load rendering data from WAD: textures, sprites, fonts, colormaps, lighting tables
2. Initialize VGA Mode-X: call `VL_SetVGAPlaneMode()` to enable 320×200×8bpp planar graphics
3. Allocate triple-buffer framebuffer in DOS extended memory
4. Clear video memory and set initial palette
5. Compute/load lookup tables: `pixelangle[]` (perspective correction), gamma curves, distance shading

**Per-frame during active gameplay:**
1. Top-level `PlayLoop()` or main game loop calls `ThreeDRefresh()`
2. Update camera state: `viewx`, `viewy`, `viewangle`, `viewsin`, `viewcos` from `player` object
3. Call `Refresh()` to execute ray-casting:
   - For each screen column: cast ray from player toward pixel; traverse tilemap with fixed-point stepping
   - Detect wall intersection; resolve wall height and texture coordinate via `CalcHeight()`
   - Handle special cases: doors (interpolate open/close frame), masked walls, multi-tile doors
   - Populate `posts[col]` with wall geometry
4. Build visible objects list: iterate actors/statics; project to screen space; depth-sort by distance
5. Render frame in back-buffer:
   - Clear or fill floor/ceiling regions
   - Render floor/ceiling horizontal spans (call `DrawRow()`, `DrawSkyPost()` assembly routines)
   - Render wall posts (iterate `posts[]`, invoke `DrawPost()` assembly routines with shading lookups)
   - Render sprites: iterate visible objects; call `ScaleShape()` or `ScaleTransparentShape()` for each sprite column
6. Composite HUD: call `DrawPlayScreen()` or similar to overlay weapons, ammo, health, messages
7. Apply transient effects: screen shake offset, damage flash (red tint via `redmap[]`), rotation angle
8. Flip framebuffer: wait for VBlank, update VGA address registers to display back-buffer; swap pointers
9. Repeat for next frame

**Shutdown:**
- Restore VGA text mode via BIOS INT 0x10
- Free allocated framebuffer memory
- Clean up WAD-resident graphics and tables

## Notable Implementation Details

- **Hierarchical ray-casting**: 4-pixel screen-space comb filter with binary subdivision reduces rays from 320 to ~80, then linearly interpolates wall height between cast rays to fill gaps—balances quality and CPU budget
- **Fixed-point arithmetic throughout**: All math uses SFRACBITS (16-bit fractional precision) to avoid floating-point ops; macros `FixedMul()`, `FixedDiv2()` from watcom.h perform 16.16 operations efficiently via inline assembly
- **Self-modifying code**: Inner-loop assembly routines (`DrawPost_`, `DrawRow_`, `TextureLine_`) patch runtime parameters (scale increments, texture addresses) directly into instruction immediates, enabling different configurations without branching
- **Planar VGA Mode-X rendering**: Exploits 4-plane graphics mode (1 bit per plane, 4-fold parallelism) via plane-mask writes; chunked rendering processes 4 pixels per write for efficiency; critical for 320×200 performance on 486-class DOS hardware
- **Masked/transparent walls**: Extends Wolf3D with DOOM-style masked column rendering for windows, platforms spanning multiple Z-levels, and multi-frame animated walls (stored as sequential lumps)
- **Sprite scaling via fixed-point**: Precomputes texture coordinate steps; interpolates per-pixel during column draw to handle fractional scaling without division per pixel
- **Distance-based lighting**: Per-tile light source array with radius-squared distance falloff; shading table lookup during post-draw applies lighting without per-pixel computation
- **Vertical blank synchronization**: All `FlipPage()` operations poll VGA status register (0x3DA) to sync with active display; prevents tearing artifacts critical on CRT hardware
- **Dirty-rectangle optimization**: Tracks 64×64-pixel blocks dirtied by updates; `VL_MemToScreen()` copies only dirty blocks to video memory, reducing bandwidth by ~70% for static scenes
- **Masked sprite rendering**: Sprites with transparency iterate column offsets; `DrawMaskedPost_` skips pixels marked as transparent (0xFF) to compose complex shapes over background
