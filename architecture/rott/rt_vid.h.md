# rott/rt_vid.h

## File Purpose
Public header for the video/graphics rendering subsystem (RT_VID.C). Declares drawing primitives, screen update functions, and palette management for a software-based tile-rendering engine. Uses a dirty-rect update buffer for efficient screen synchronization.

## Core Responsibilities
- **Screen drawing primitives**: Direct pixel/memory-to-screen transfers, rectangles, lines, and picture blitting
- **Tile-based rendering**: Tiled region drawing with offset support
- **Palette management**: Setting/getting colors, fade effects, palette switching, LBM decompression
- **Dirty-rect updates**: Screen block marking and lazy screen refresh
- **Texture/border effects**: Textured lines/bars, border color management, fade-to-color transitions

## Key Types / Data Structures
None defined in this file. Depends on `lumpy.h`:
| Name | Kind | Purpose |
|------|------|---------|
| pic_t | struct | Simple picture (width, height, pixel data) |
| lbm_t | struct | LBM image with embedded 768-byte palette |
| patch_t | struct | Sprite/patch with column-offset rendering |
| font_t | struct | Font glyph data with character metrics |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| updateptr | byte* | extern | Current write pointer into update buffer |
| mapwidthtable | unsigned[64] | extern | Precomputed width lookup table |
| uwidthtable | unsigned[UPDATEHIGH] | extern | Update row width table |
| blockstarts | unsigned[260] | extern | Block start offsets (UPDATEWIDE × UPDATEHIGH = 20×13) |
| update | byte[260] | extern | Dirty-rect update buffer for screen regions |
| screenfaded | boolean | extern | Flag tracking fade state |

## Key Functions / Methods

### VW_MarkUpdateBlock
- **Signature:** `int VW_MarkUpdateBlock(int x1, int y1, int x2, int y2)`
- **Purpose:** Mark a rectangular region for screen update
- **Inputs:** Bounding box coordinates (x1, y1, x2, y2)
- **Outputs/Return:** Region index or status code
- **Notes:** Implements dirty-rect tracking; called before drawing operations

### VW_UpdateScreen
- **Signature:** `void VW_UpdateScreen(void)`
- **Purpose:** Flush marked screen regions to display
- **Notes:** Main refresh function; syncs all marked dirty rectangles

### VL_MemToScreen
- **Signature:** `void VL_MemToScreen(byte *source, int width, int height, int x, int y)`
- **Purpose:** Copy raw pixel buffer to screen at (x,y)
- **Inputs:** Source pixel data, dimensions, destination coordinates
- **Notes:** Low-level blit; used by higher-level drawing functions

### DrawTiledRegion
- **Signature:** `void DrawTiledRegion(int x, int y, int width, int height, int offx, int offy, pic_t *tile)`
- **Purpose:** Draw repeating tile pattern with offset
- **Inputs:** Region bounds, offset within tile, tile picture
- **Notes:** Core tilemap rendering function

### VWB_DrawPic / VL_Bar / VWB_Bar
- **Purpose:** Draw picture and filled rectangles (buffered vs. direct variants)
- **Notes:** VWB_ prefix typically indicates "Video With Buffer"

### VL_FadeOut / VL_FadeIn / VL_FadeToColor
- **Purpose:** Animated palette transitions
- **Inputs:** Color range, target RGB, transition steps
- **Notes:** Supports partial palette fades (start/end range); MenuFadeOut/In macros for standard black fades

### VL_SetColor / VL_GetColor / VL_FillPalette
- **Purpose:** Direct palette manipulation
- **Notes:** Palette is 256 entries × 3 bytes (RGB); operates on DAC hardware or in-memory palette

### VL_DecompressLBM
- **Signature:** `void VL_DecompressLBM(lbm_t *lbminfo, boolean flip)`
- **Purpose:** Decompress LBM image and load embedded palette
- **Inputs:** LBM structure, flip flag for vertical flip
- **Notes:** Handles palette from lbm_t.palette array

## Control Flow Notes
- **Initialization**: VL_DecompressLBM loads title screens; SetBorderColor configures borders
- **Per-frame rendering**: Game loop calls drawing functions, then VW_MarkUpdateBlock, then VW_UpdateScreen
- **Palette effects**: Menu/fade sequences use VL_FadeOut/In macros and SwitchPalette
- **Tile rendering**: Main game view rendered via DrawTiledRegion in viewport region

## External Dependencies
- **lumpy.h**: pic_t, lpic_t, font_t, lbm_t, patch_t, transpatch_t, cfont_t typedefs
- **C primitives**: byte, boolean, int, unsigned, short, char
- **Implied**: Video hardware access (VGA DAC, framebuffer) defined elsewhere
