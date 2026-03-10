# rott/rt_build.c

## File Purpose
Implements a 3D menu rendering system that projects textured planes in perspective and renders 2D UI elements (text, sprites, primitives) onto a double-buffered menu surface. Provides the backbone for animated menu screens with rotating 3D backgrounds.

## Core Responsibilities
- Menu buffer lifecycle (initialization, clearing, shutdown)
- 3D-to-2D perspective projection for planar geometry
- Depth-sorted plane rendering with affine texture mapping
- Double-buffering infrastructure for smooth menu transitions
- Drawing primitives (boxes, lines, pixels)
- Text rendering with multiple shading and intensity modes
- Sprite and picture composition onto the menu buffer
- Menu animation and flip transitions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `plane_t` | struct | 3D plane with position, texture, dimensions; defined in `planelist` |
| `visobj_t` | struct | Transformed plane in screen space with height/position; from rt_draw.h |
| `patch_t` | struct | Sprite/patch with column offsets and dimensions; loaded from WAD |
| `pic_t` | struct | Picture format with width/height and packed pixel data |
| `font_t` | struct | Font metadata with character widths and offsets |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `intensitytable` | byte* | global | Color blending lookup table for intensity/shading effects |
| `menubuf` | byte* | static | Active menu buffer pointer (points to menubuffers[0] or [1]) |
| `menubuffers[2]` | byte*[2] | static | Two buffers for double-buffering menu animation |
| `menutitles[2][40]` | char[2][40] | static | Title strings for each buffer |
| `alternatemenubuf` | int | static | Index toggle (0 or 1) for active buffer |
| `titleshade` | int | static | Current shade value for title (animates 10–22 range) |
| `titleshadedir` | int | static | Direction multiplier for title shade animation (±1) |
| `titleyoffset` | int | static | Y-offset for title during flip animation |
| `titlestring[40]` | char[40] | static | Currently displayed title string |
| `readytoflip` | int | static | Flag: buffer swap queued and ready to execute |
| `MenuBufStarted` | boolean | static | Guard flag: menu buffer initialized |
| `mindist` | int | static | Clipping distance for 3D projection (0x2700) |
| `BackgroundDrawn` | boolean | static | Caching flag: background copied to hidden buffer |
| `planelist[MAXPLANES]` | plane_t[] | static | Array of 3D planes to render |
| `planeptr` | plane_t* | static | Current insertion point in plane list |
| `StringShade` | int | static | Shade index for string rendering (uninitialized declaration) |

## Key Functions / Methods

### SetupMenuBuf
- **Signature:** `void SetupMenuBuf(void)`
- **Purpose:** Initialize menu buffer system; allocate double buffers, reset plane list, configure initial 3D scene geometry
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates memory via `SafeMalloc`, modifies global state (`MenuBufStarted`, `menubuf`, `planelist`, `planeptr`, `BackgroundDrawn`)
- **Calls:** `SafeMalloc`, `ClearMenuBuf`, `NextPlaneptr`, `W_GetNumForName`
- **Notes:** Guard check prevents re-initialization; sets up 4 planes forming a box around menu space; calls `ClearMenuBuf` to initialize background

### PositionMenuBuf
- **Signature:** `void PositionMenuBuf(int angle, int distance, boolean drawbackground)`
- **Purpose:** Render menu at camera angle/distance; perform 3D transform, draw planes, text, optionally cache background
- **Inputs:** `angle` (camera rotation), `distance` (camera depth from origin), `drawbackground` (cache to hidden buffer flag)
- **Outputs/Return:** None
- **Side effects:** Modifies `posts[]` array (screen-space plane data), calls `FlipPage` (video buffer swap), reads/writes display memory
- **Calls:** `CalcTics`, `SetupPlanes`, `VL_DrawPostPic`, `CalcPlanes`, `DrawTransformedPlanes`, `DrawPlanePosts`, `W_CacheLumpName`, `US_MeasureStr`, `US_ClippedPrint`, `FlipPage`, `VL_CopyDisplayToHidden`
- **Notes:** Title shade oscillates ±6 units around 16; background caching allows reuse across frames; uses fixed-point math for trig

### CalcPlanes
- **Signature:** `void CalcPlanes(int px, int py, int angle)`
- **Purpose:** Transform all planes in `planelist` to camera-relative coordinates; populate `vislist` with visible geometry
- **Inputs:** `px`, `py` (camera position in world), `angle` (camera facing)
- **Outputs/Return:** None
- **Side effects:** Resets `visptr`, populates `vislist[0..numvisible]` with transformed planes
- **Calls:** `ResetVisptr`, `SideOfLine`, `GetPoint`, `NextVisptr`
- **Notes:** Culls planes facing away via `SideOfLine` test; reverses winding if back-facing

### GetPoint
- **Signature:** `void GetPoint(int x1, int y1, int px, int py, int *screenx, int *height, int angle)`
- **Purpose:** Project single 3D point to 2D screen coordinates and compute on-screen height
- **Inputs:** `(x1,y1)` world point, `(px,py)` camera position, `angle` camera facing
- **Outputs/Return:** `*screenx` (0–319 screen column), `*height` (column height in pixels)
- **Side effects:** None
- **Calls:** `FixedMul`, `costable[]`, `sintable[]`
- **Notes:** Clamps `nx` (depth) to `mindist` to prevent division overflow; screen X clamped to [0, 319]

### DrawTransformedPlanes
- **Signature:** `void DrawTransformedPlanes(void)`
- **Purpose:** Depth-sort visible planes by height; render back-to-front
- **Inputs:** None (reads from `vislist`, `visptr`)
- **Outputs/Return:** None
- **Side effects:** Modifies `posts[]` array (column heights), zeros `viewheight` on rendered planes
- **Calls:** `InterpolatePlane`
- **Notes:** Simple painter's algorithm (sorts by max height each frame); O(n²) but `n` is typically 4–10 planes

### InterpolatePlane
- **Signature:** `void InterpolatePlane(visobj_t *plane)`
- **Purpose:** Affine-map plane texture across screen columns; populate `posts[]` with texture/height data
- **Inputs:** `plane` transformed plane object
- **Outputs/Return:** None
- **Side effects:** Modifies `posts[i].wallheight`, `.texture`, `.lump`, `.offset` for columns in plane's screen range
- **Calls:** None
- **Notes:** Fixed-point perspective math; skips columns where plane is occluded or height ≤ 0; texture coordinate division by `bot` may be unsafe if `bot==0`

### DrawPlanePosts
- **Signature:** `void DrawPlanePosts(void)`
- **Purpose:** Render all columns in `posts[]` array by fetching textures and calling assembly draw routines
- **Inputs:** None (reads `posts[]`)
- **Outputs/Return:** None
- **Side effects:** Writes to video memory via `VGAWRITEMAP`, assembly functions
- **Calls:** `VGAWRITEMAP`, `W_CacheLumpNum`, `DrawRotPost`
- **Notes:** Caches last wall lump to avoid redundant WAD fetches; operates on 4 video planes (planar VGA mode)

### DrawRotPost
- **Signature:** `void DrawRotPost(int height, byte *src, byte *buf, int origheight)`
- **Purpose:** Scale and draw a single textured column to buffer
- **Inputs:** `height` output column height, `src` texture column data, `buf` destination buffer, `origheight` original texture height
- **Outputs/Return:** None
- **Side effects:** Sets globals `hp_srcstep`, `hp_startfrac`; calls assembly `DrawMenuPost`
- **Calls:** `DrawMenuPost`
- **Notes:** Handles Y clipping; computes fixed-point source step and start offset

### FlipMenuBuf
- **Signature:** `void FlipMenuBuf(void)`
- **Purpose:** Animate transition from current to alternate buffer; rotate title and perform flip sound/effect
- **Inputs:** None (reads `Menuflipspeed`, `readytoflip`)
- **Outputs/Return:** None
- **Side effects:** Swaps buffers, animates title Y offset and rotation angle, plays sound, calls `PositionMenuBuf` multiple times
- **Calls:** `PositionMenuBuf`, `RefreshMenuBuf`, `MN_PlayMenuSnd`
- **Notes:** If `Menuflipspeed ≤ 5`, instant swap; else animates over `Menuflipspeed–5` ticks with 180° rotation and sound effect

## Text Rendering Functions

**DrawMenuBufPropString** / **DrawMenuBufIString** / **DrawTMenuBufPropString**: Render font glyphs to menu buffer with optional shading (unshaded, intensity-based, or table-lookup). Use `CurrentFont` or `IFont` metadata.

**MenuBufPrint** / **MenuBufCPrint** / **MenuBufPrintLine** / **MenuTBufPrintLine**: High-level text output with window positioning, newline support, and optional centering.

## Drawing Primitives

**DrawMenuBufItem** / **DrawIMenuBufItem** / **DrawTMenuBufItem** / **DrawColoredMenuBufItem**: Render sprites with offset-encoded columns; support intensity and player-color shading.

**DrawMenuBufPic** / **DrawTMenuBufPic**: Render packed planar pictures.

**DrawTMenuBufBox** / **DrawTMenuBufHLine** / **DrawTMenuBufVLine**: Render shaded primitives for UI borders/lines.

## Control Flow Notes
1. **Init:** `SetupMenuBuf()` → allocate buffers, configure plane geometry
2. **Frame loop:** `PositionMenuBuf()` → 3D transform (`CalcPlanes`, `DrawTransformedPlanes`, `DrawPlanePosts`) → 2D UI (draw items/text) → `FlipPage()`
3. **Transition:** `SetAlternateMenuBuf()` + `ClearMenuBuf()` → draw new UI → `FlipMenuBuf()` (animated swap)
4. **Shutdown:** `ShutdownMenuBuf()` → deallocate buffers

## External Dependencies
- **Includes:** `RT_DEF.H` (constants, types), `rt_draw.h` (visobj_t, drawing state), `watcom.h` (fixed-point math), `lumpy.h` (WAD structures), `w_wad.h` (resource loading), `rt_util.h`, `rt_vid.h`, `rt_sound.h`, `modexlib.h` (video/sound)
- **Extern symbols (defined elsewhere):**
  - `costable[]`, `sintable[]`: precomputed trig lookup tables
  - `W_CacheLumpNum()`, `W_CacheLumpName()`, `W_GetNumForName()`: WAD resource management
  - `posts[]`: per-column screen state (rt_draw.c)
  - `colormap`, `playermaps[]`: color lookup tables
  - `Keyboard[]`: input state array
  - `Menuflipspeed`: configuration variable
  - `CurrentFont`, `IFont`: active font pointers
  - `tics`, `viewwidth`, `viewheight`, `centery`, etc.: rendering state from rt_draw.c
  - `ylookup[]`: scanline offset table
  - `bufferofs`, `screenofs`: video memory pointers
  - `PrintX`, `PrintY`: text cursor state
  - `shadingtable`: active shading palette (local and extern)
