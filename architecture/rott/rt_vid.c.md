# rott/rt_vid.c

## File Purpose
Core video/graphics subsystem for VGA Mode-X rendering. Implements double-buffered screen updates via dirty rectangles, drawing primitives (bars, lines, pictures), palette management, fade/transition effects, and LBM image decompression. Directly interfaces with VGA hardware via planar graphics mode.

## Core Responsibilities
- **VGA Mode-X rendering**: Planar graphics memory access with per-plane writes
- **Double-buffering**: Update block tracking to minimize screen transfers
- **Drawing primitives**: Bars, lines (horizontal/vertical/arbitrary), pictures, tiled regions
- **Palette operations**: Fill, set, get color; fade/fade-in/transition effects
- **Image handling**: LBM decompression and screen output
- **Screen management**: Border color control, update flushing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| pic_t | struct (lumpy.h) | Image sprite: width, height, packed pixel data |
| lbm_t | struct (lumpy.h) | LBM image: height, width, palette (768 bytes), compressed data |
| pixmasks | static byte[4] | VGA plane bit masks: {1,2,4,8} for pixel selection |
| leftmasks | static byte[4] | Left edge masks for partial-byte writes: {15,14,12,8} |
| rightmasks | static byte[4] | Right edge masks for partial-byte writes: {1,3,7,15} |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| updateptr | byte* | global | Pointer to update block dirty-rectangle flag buffer |
| mapwidthtable | unsigned[64] | global | Width lookup table for tile-based coordinates |
| uwidthtable | unsigned[UPDATEHIGH] | global | Update block row offset table (20 wide × 13 high) |
| blockstarts | unsigned[UPDATEWIDE×UPDATEHIGH] | global | Starting addresses for each update block |
| update | byte[UPDATESIZE] | global | Update flags (1=block needs refresh to display) |
| palette1 | byte[256][3] | global | Current palette (256 colors × 3 channels RGB) |
| palette2 | byte[256][3] | global | Intermediate palette for fade transitions |
| screenfaded | boolean | global | Flag: true when screen is fully faded to color |
| pixmasks, leftmasks, rightmasks | static arrays | static | VGA bit masks for planar graphics writes |

## Key Functions / Methods

### VL_MemToScreen
- **Signature:** `void VL_MemToScreen(byte *source, int width, int height, int x, int y)`
- **Purpose:** Copy memory buffer to screen via all 4 VGA planes, handling pixel-to-plane conversion at arbitrary X positions.
- **Inputs:** source buffer, width (pixels), height (pixels), x/y destination on screen
- **Outputs/Return:** None (modifies screen memory)
- **Side effects:** Writes to VGA memory at `bufferofs+ylookup[y]+(x>>2)` on each plane; sets `VGAMAPMASK` register
- **Calls:** `VGAMAPMASK`, `memcpy`
- **Notes:** Assumes source is packed horizontally; iterates 4 planes with shifting mask. Width in pixels; destination address shifts by 2 bits (x>>2). No bounds checking.

### DrawTiledRegion
- **Signature:** `void DrawTiledRegion(int x, int y, int width, int height, int offx, int offy, pic_t *tile)`
- **Purpose:** Fill rectangular region with repeating tile image, applying offset for pattern phase.
- **Inputs:** Region (x,y,width,height), tile offset (offx,offy), tile image pointer
- **Outputs/Return:** None (modifies screen)
- **Side effects:** Writes to `bufferofs` for all 4 VGA planes; sets mask register per plane
- **Calls:** `VGAMAPMASK`
- **Notes:** Wraps tile coordinates when offset exceeds tile bounds. Processes per-plane; width is in bytes (>>2). Inner loops tile both X and Y.

### VWB_DrawPic
- **Signature:** `void VWB_DrawPic(int x, int y, pic_t *pic)`
- **Purpose:** Draw image at (x,y) and mark affected region for screen update.
- **Inputs:** x/y coordinates, pic pointer
- **Outputs/Return:** None
- **Side effects:** Calls `VW_MarkUpdateBlock`, then `VL_MemToScreen` if block marked
- **Calls:** `VW_MarkUpdateBlock`, `VL_MemToScreen`
- **Notes:** Conditional draw (only if mark succeeds); width scaled by 4 (pic->width<<2).

### VL_Bar
- **Signature:** `void VL_Bar(int x, int y, int width, int height, int color)`
- **Purpose:** Draw solid filled rectangle with given color, handling partial-byte edges via masks.
- **Inputs:** x/y origin, width/height (pixels), color index (0–255)
- **Outputs/Return:** None
- **Side effects:** Writes to VGA memory; sets `VGAMAPMASK` per edge segment
- **Calls:** `VGAMAPMASK`, `memset`
- **Notes:** Optimizes for single-byte case (midbytes<0). Left/right edges handled with partial masks; middle bytes filled with `memset`. Edge masks from static `leftmasks`/`rightmasks` tables based on x&3 alignment.

### VL_TBar
- **Signature:** `void VL_TBar(int x, int y, int width, int height)`
- **Purpose:** Draw transparent/darkened rectangle by remapping pixels through colormap.
- **Inputs:** x/y origin, width/height
- **Outputs/Return:** None
- **Side effects:** Reads pixel via `VGAREADMAP`, writes remapped pixel; increments plane mask
- **Calls:** `VGAREADMAP`, `VGAMAPMASK`
- **Notes:** Uses `colormap+(27<<8)` offset for darkening (shade table 27). Manually advances plane mask and wrapped coordinates; plane 4 iterations.

### VW_MarkUpdateBlock
- **Signature:** `int VW_MarkUpdateBlock(int x1, int y1, int x2, int y2)`
- **Purpose:** Convert pixel bounds to update block grid, mark tiles for refresh, return success.
- **Inputs:** Bounding box (x1,y1) to (x2,y2) in pixels
- **Outputs/Return:** 1 if block intersects display, 0 if fully off-screen
- **Side effects:** Sets `update[]` bytes to 1 for affected tiles; reads/modifies `updateptr`
- **Calls:** None (direct array access)
- **Notes:** Clamps tile coordinates to valid range [0, UPDATEWIDE/UPDATEHIGH). Tile size is 2^PIXTOBLOCK pixels (16 pixels). Uses `uwidthtable` for row offsets. Returns early (0) if bounds completely outside display.

### VL_FadeOut
- **Signature:** `void VL_FadeOut(int start, int end, int red, int green, int blue, int steps)`
- **Purpose:** Fade palette colors (start–end) to target color over N steps, blocking until complete.
- **Inputs:** Palette range (start–end entry index), target RGB, step count
- **Outputs/Return:** None
- **Side effects:** Reads palette via `VL_GetPalette`, interpolates in `palette1`/`palette2`, writes via `VL_SetPalette` each step, sets `screenfaded=true`
- **Calls:** `WaitVBL`, `VL_GetPalette`, `VL_SetPalette`, `VL_FillPalette`
- **Notes:** Early exit if already faded. Interpolates each RGB channel separately: `new = orig + (target - orig) * i / steps`. Final frame fills palette with target color.

### VL_FadeIn
- **Signature:** `void VL_FadeIn(int start, int end, byte *palette, int steps)`
- **Purpose:** Fade from current palette to target palette over N steps.
- **Inputs:** Palette range, target palette data, step count
- **Outputs/Return:** None
- **Side effects:** Reads current palette, copies to working buffer, interpolates per step, sets `screenfaded=false`
- **Calls:** `WaitVBL`, `VL_GetPalette`, `VL_SetPalette`
- **Notes:** Treats palette as flat byte array (start/end multiplied by 3 for RGB). Interpolates difference: `new = current + (target - current) * i / steps`.

### VL_FadeToColor
- **Signature:** `void VL_FadeToColor(int time, int red, int green, int blue)`
- **Purpose:** Fade entire palette to solid color over time milliseconds, updating 3D view dynamically.
- **Inputs:** Time (milliseconds), target RGB
- **Outputs/Return:** None
- **Side effects:** Modifies `maxshade`/`minshade` globals, calls `ThreeDRefresh()` and `CalcTics()` each frame, sets `screenfaded=true`
- **Calls:** `WaitVBL`, `VL_GetPalette`, `VL_SetPalette`, `FixedMul`, `ThreeDRefresh`, `CalcTics`
- **Notes:** Driven by `tics` (game time delta). Uses fixed-point math (<<16, /time, FixedMul) for precision. Darkens shade boundaries as color fades. Calls 3D refresh mid-fade for cinematic effect.

### VL_DecompressLBM
- **Signature:** `void VL_DecompressLBM(lbm_t *lbminfo, boolean flip)`
- **Purpose:** Decompress LBM image (RLE) to screen buffer, set palette, optionally update display.
- **Inputs:** LBM info struct, flip flag (update screen on true)
- **Outputs/Return:** None
- **Side effects:** Allocates 64KB temp buffer, decompresses to `bufferofs`, sets palette via `VL_FadeIn`, marks update block
- **Calls:** `SafeMalloc`, `VL_ClearBuffer`, `VL_NormalizePalette`, `VW_MarkUpdateBlock`, `SafeFree`, `VW_UpdateScreen`, `VL_FadeIn`
- **Notes:** RLE codes: >0x80 = run of repeated byte; <0x80 = literal run; 0x80 = NOP. Decompresses to planar layout (all plane 0, then 1, etc.). Assumes 320×200 output.

### VL_DrawLine
- **Signature:** `void VL_DrawLine(int x1, int y1, int x2, int y2, byte color)`
- **Purpose:** Draw arbitrary line from (x1,y1) to (x2,y2) using fixed-point rasterization.
- **Inputs:** Endpoints, color index
- **Outputs/Return:** None
- **Side effects:** Writes pixels to `bufferofs` via `VGAWRITEMAP` for each step
- **Calls:** `VGAWRITEMAP`, `abs`
- **Notes:** Uses fixed-point math (<<16) to avoid floating-point. Chooses major axis (dy or dx). Increments are scaled: `xinc/yinc = (dx/dy)<<16 / count`. Walks major axis and interpolates minor axis.

### DrawXYPic
- **Signature:** `void DrawXYPic(int x, int y, int shapenum)`
- **Purpose:** Draw sprite from lump at (x,y), with bounds checking.
- **Inputs:** x/y screen coordinates, sprite lump number
- **Outputs/Return:** None
- **Side effects:** Caches lump, writes to `bufferofs` via `VGAWRITEMAP` for each plane
- **Calls:** `W_CacheLumpNum`, `Error`
- **Notes:** Errors if (x,y) or (x+width,y+height) out of 320×200 bounds. Processes 4 planes; source bytes read sequentially.

## Control Flow Notes
**Initialization phase:** Update block system initialized by external code (updateptr, uwidthtable, blockstarts populated).

**Frame update loop:**
1. Game draws to `bufferofs` (back buffer) using VWB_* functions
2. VWB_* functions call `VW_MarkUpdateBlock` to flag dirty tiles
3. At frame end, `VW_UpdateScreen()` calls `VH_UpdateScreen()` to copy marked blocks to `displayofs`

**Palette transitions:** Called between scene loads; block until fade completes (e.g., menu fade-in/out).

**Screen setup:** LBM decompression and VL_DrawPostPic used for splash screens/level start images.

## External Dependencies
- **System includes:** `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<dos.h>`, `<conio.h>` (DOS/Watcom)
- **Key local headers:** 
  - `rt_def.h` (constants, types: UPDATEWIDE, UPDATEHIGH, UPDATESIZE)
  - `_rt_vid.h` (private macros: PIXTOBLOCK, VW_Hlin/Vlin wrappers)
  - `lumpy.h` (pic_t, lbm_t struct definitions)
  - `modexlib.h` (VGA low-level, likely VGAMAPMASK, VGAREADMAP, VGAWRITEMAP, SCREENBWIDE)
  - `rt_view.h` (VH_UpdateScreen, ThreeDRefresh, CalcTics)
  - `rt_util.h` (SafeMalloc, SafeFree, WaitVBL, VL_GetPalette, VL_SetPalette, FixedMul)
  - `w_wad.h` (W_CacheLumpNum)
- **External symbols** (defined elsewhere):
  - `bufferofs`, `displayofs` (screen buffer offsets)
  - `ylookup[]` (Y-to-offset lookup table)
  - `linewidth` (screen width in bytes)
  - `colormap` (color remapping tables for transparency/shading)
  - `maxshade`, `minshade` (shade table extents)
  - `tics` (elapsed game ticks since last frame)
