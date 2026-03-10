# rott/rt_scale.c

## File Purpose
Handles scaling, transformation, and rendering of sprites in the ROTT software-rasterized 3D engine. Implements column-based sprite drawing with support for transparency, masking, light shading, and variable scaling for perspective-correct rendering.

## Core Responsibilities
- Calculate and apply light/shading tables based on player position and object height
- Scale and render individual sprite columns (posts) at various transparency/masking modes
- Composite scaled sprite shapes into the VGA framebuffer with occlusion testing
- Render weapon/HUD sprites with custom scaling
- Support both scaled and unscaled sprite drawing for UI elements
- Handle VGA planar mode rendering across multiple pixel planes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `visobj_t` | struct | Visible sprite object with position, scale, shape reference, color mapping (defined in rt_draw.h) |
| `patch_t` | struct | Sprite shape data with column offsets and dimensions (from lumpy.h) |
| `transpatch_t` | struct | Transparent sprite shape variant with per-pixel alpha hints |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `dc_texturemid` | int | static | Texture middle Y coordinate for scaling calculations |
| `dc_iscale` | int | static | Integer scale factor (1/invscale); texture U per screen pixel |
| `dc_invscale` | int | static | Inverse scale; screen pixels per texture U |
| `sprtopoffset` | int | static | Top Y offset in screen space for sprite column rendering |
| `dc_yl` / `dc_yh` | int | static | Draw column Y bounds (low/high) after clipping |
| `dc_source` | byte* | static | Current source sprite data pointer |
| `centeryclipped` | int | static | Clipped center Y coordinate for weapon/HUD drawing |
| `transparentlevel` | int | static | Translucency level (0–63) for transparent post rendering |
| `shadingtable` | byte* | static | Current palette lookup table for light/color mapping |

## Key Functions / Methods

### SetPlayerLightLevel
- **Signature:** `void SetPlayerLightLevel(void)`
- **Purpose:** Determine shading table based on player's height and world lighting at player position.
- **Inputs:** None (reads global `player`, `MISCVARS`, lighting state)
- **Outputs/Return:** None (sets global `shadingtable`)
- **Side effects:** Modifies `shadingtable` pointer; checks gas mask/fulllight modes.
- **Calls:** `LightSourceAt()` (macro), conditional lookup.
- **Notes:** Intercept calculation uses player angle and coordinate bits for selective lighting. `whereami=23` debug marker.

### SetLightLevel
- **Signature:** `void SetLightLevel(int height)`
- **Purpose:** Calculate shading table based on object height, applying distance-based darkness.
- **Inputs:** `height` — object's world height
- **Outputs/Return:** None (sets global `shadingtable`)
- **Side effects:** Modifies `shadingtable` pointer; respects fulllight/fog/gas mask modes.
- **Calls:** None
- **Notes:** Simple linear height-to-shade mapping with clamping. `whereami=24`.

### ScaleTransparentPost
- **Signature:** `void ScaleTransparentPost(byte *src, byte *buf, int level)`
- **Purpose:** Draw a scaled, translucent sprite column with per-pixel color lookup and alpha blending.
- **Inputs:** `src` — sprite post data (run-length encoded offset/length pairs), `buf` — framebuffer pointer, `level` — transparency level (0–63)
- **Outputs/Return:** None (writes to `buf`)
- **Side effects:** Modifies global `dc_yl`, `dc_yh`, `dc_source`, `shadingtable`. Calls `R_TransColumn()` and `R_DrawColumn()`.
- **Calls:** `R_TransColumn()`, `R_DrawColumn()` (defined elsewhere, likely assembly)
- **Notes:** Post data is RLE-encoded with magic value 254 signaling a transparent pixel run. Uses `dc_invscale` for Y mapping. `whereami=25`. Dev-mode validation of shadingtable and level bounds.

### ScaleMaskedPost
- **Signature:** `void ScaleMaskedPost(byte *src, byte *buf)`
- **Purpose:** Draw a scaled opaque sprite column with masking.
- **Inputs:** `src` — sprite post data, `buf` — framebuffer pointer
- **Outputs/Return:** None
- **Side effects:** Modifies global column drawing state, calls `R_DrawColumn()`.
- **Calls:** `R_DrawColumn()`
- **Notes:** Simpler than `ScaleTransparentPost`; no alpha blending. `whereami=26`.

### ScaleClippedPost
- **Signature:** `void ScaleClippedPost(byte *src, byte *buf)`
- **Purpose:** Draw a scaled column with clipping (differs slightly in Y calculation from masked version).
- **Inputs:** `src`, `buf` — sprite data and framebuffer
- **Outputs/Return:** None
- **Side effects:** Similar to masked post; calls `R_DrawClippedColumn()`.
- **Calls:** `R_DrawClippedColumn()`
- **Notes:** Uses `(topscreen+SFRACUNIT-1)` for Y low, suggesting tighter clipping. `whereami=27`.

### ScaleSolidMaskedPost
- **Signature:** `void ScaleSolidMaskedPost(int color, byte *src, byte *buf)`
- **Purpose:** Draw a sprite column with a single solid color (ignoring source palette).
- **Inputs:** `color` — fill color index, `src` — post data (for geometry), `buf` — framebuffer
- **Outputs/Return:** None
- **Side effects:** Calls `R_DrawSolidColumn()`.
- **Calls:** `R_DrawSolidColumn()`
- **Notes:** Used for colored overlays (e.g., player pain flash). `whereami=28`.

### ScaleTransparentClippedPost
- **Signature:** `void ScaleTransparentClippedPost(byte *src, byte *buf, int level)`
- **Purpose:** Combination of transparent and clipped post rendering.
- **Inputs:** `src`, `buf`, `level` — sprite data, buffer, transparency
- **Outputs/Return:** None
- **Side effects:** Modifies shading table; calls `R_TransColumn()` and `R_DrawClippedColumn()`.
- **Calls:** `R_TransColumn()`, `R_DrawClippedColumn()`
- **Notes:** `whereami=29`.

### ScaleMaskedWidePost / ScaleClippedWidePost
- **Signature:** `void ScaleMaskedWidePost(byte *src, byte *buf, int x, int width)` (and clipped variant)
- **Purpose:** Draw sprite posts spanning multiple VGA planar modes (handling 4-plane interleaving).
- **Inputs:** `src` — sprite data, `buf` — framebuffer, `x` — X position, `width` — column width
- **Outputs/Return:** None
- **Side effects:** Calls `VGAMAPMASK()`, `VGAWRITEMAP()` to switch VGA planes; calls underlying post functions multiple times.
- **Calls:** `VGAMAPMASK()`, `VGAWRITEMAP()`, `ScaleMaskedPost()` / `ScaleClippedPost()` (3 invocations for 3 planes)
- **Notes:** Uses `mapmasks1/2/3` lookup tables. `whereami=30/31`.

### ScaleShape
- **Signature:** `void ScaleShape(visobj_t *sprite)`
- **Purpose:** Scale and render a complete opaque sprite at world position based on distance (perspective).
- **Inputs:** `sprite` — visible object with position, distance, shape reference
- **Outputs/Return:** None
- **Side effects:** Extensive: sets all drawing globals (`dc_invscale`, `dc_iscale`, `dc_texturemid`, `sprtopoffset`, `shadingtable`), modifies `posts[]` array implicitly via wall height checks, writes to framebuffer.
- **Calls:** `W_CacheLumpNum()`, `FixedMul()`, `ScaleMaskedWidePost()` or `ScaleMaskedPost()` (in loop)
- **Notes:** Handles both high-detail (wide post) and low-detail (narrow post) rendering paths. Skips columns where wall occludes sprite. Culls off-screen. `whereami=32`.

### ScaleTransparentShape
- **Signature:** `void ScaleTransparentShape(visobj_t *sprite)`
- **Purpose:** Scale and render a transparent sprite with per-pixel blending.
- **Inputs:** `sprite` — visible object
- **Outputs/Return:** None
- **Side effects:** Similar to `ScaleShape`; calls `ScaleTransparentPost()` instead of masked variant.
- **Calls:** `W_CacheLumpNum()`, `FixedMul()`, `ScaleTransparentPost()` (in loop), `VGAWRITEMAP()`, `VGAREADMAP()`
- **Notes:** `whereami=33`.

### ScaleSolidShape
- **Signature:** `void ScaleSolidShape(visobj_t *sprite)`
- **Purpose:** Render a sprite with a single solid color, using shape geometry for masking.
- **Inputs:** `sprite` — visible object
- **Outputs/Return:** None
- **Side effects:** Calls `ScaleSolidMaskedPost()` in loop.
- **Calls:** `W_CacheLumpNum()`, `FixedMul()`, `ScaleSolidMaskedPost()`
- **Notes:** Used for colored pain/damage overlays. `whereami=34`.

### ScaleWeapon
- **Signature:** `void ScaleWeapon(int xoff, int y, int shapenum)`
- **Purpose:** Draw a weapon sprite on HUD with screen-space positioning and custom scaling.
- **Inputs:** `xoff` — X offset from center, `y` — Y offset, `shapenum` — sprite resource ID
- **Outputs/Return:** None
- **Side effects:** Sets light level, scales and clips to screen bounds, writes to framebuffer via `ScaleClippedPost()`.
- **Calls:** `SetPlayerLightLevel()`, `W_CacheLumpNum()`, `FixedMul()`, `ScaleClippedPost()`
- **Notes:** Uses `weaponscale` global for HUD sizing. `whereami=35`.

### DrawUnScaledSprite
- **Signature:** `void DrawUnScaledSprite(int x, int y, int shapenum, int shade)`
- **Purpose:** Draw sprite at 1:1 scale at absolute screen position.
- **Inputs:** `x`, `y` — screen coordinates, `shapenum` — sprite ID, `shade` — light level (0–31)
- **Outputs/Return:** None
- **Side effects:** Sets `dc_invscale=0x10000` (no scaling), writes to framebuffer.
- **Calls:** `W_CacheLumpNum()`, `ScaleClippedPost()`
- **Notes:** Used for HUD elements and status screens. `whereami=36`.

### DrawScreenSprite / DrawPositionedScaledSprite / DrawScreenSizedSprite
- **Signature:** `void DrawScreenSprite(int x, int y, int shapenum)` (and variants)
- **Purpose:** Variants for rendering UI/menu sprites at screen space with different scaling modes.
- **Inputs:** Vary (position, size, shape reference)
- **Outputs/Return:** None
- **Side effects:** Delegate to `ScaleWeapon()` or specialized variants; set `shadingtable` to fulllight mode.
- **Calls:** `ScaleWeapon()`, `W_CacheLumpNum()`, `FixedMul()`, `ScaleClippedPost()` / `ScaleTransparentClippedPost()`
- **Notes:** Used for menus, UI, and full-screen overlays. `whereami=37–39`.

### DrawNormalPost / DrawNormalSprite
- **Signature:** `void DrawNormalPost(byte *src, byte *buf)` and `void DrawNormalSprite(int x, int y, int shapenum)`
- **Purpose:** Legacy/fallback unscaled sprite drawing; used for menu and status rendering.
- **Inputs:** `src`/`buf` or `x`/`y`/`shapenum`
- **Outputs/Return:** None
- **Side effects:** Writes directly to framebuffer via `ylookup[]` table (no VGA plane switching for normal drawing).
- **Calls:** `W_CacheLumpNum()`, `DrawNormalPost()`
- **Notes:** Different rendering path; uses `ylookup[]` for linear Y addressing. `whereami=40–41`. Validates bounds with `Error()`.

## Control Flow Notes
This file is central to the **render phase** of the 3D refresh loop:
1. `ThreeDRefresh()` identifies visible sprites and populates `vislist[]`.
2. Back-to-front or front-to-back iteration calls `ScaleShape()`, `ScaleTransparentShape()`, `ScaleSolidShape()` for world objects.
3. Weapon/HUD rendering via `ScaleWeapon()` and `DrawScreenSprite()` happens after 3D world.
4. Menu/UI overlays use `DrawScreenSizedSprite()` and `DrawNormalSprite()`.
5. Occlusion via `posts[x].wallheight` comparison; culling for off-screen sprites.

## External Dependencies
- **rt_draw.h** — `visobj_t`, shading table, light lookup macros, drawing externs
- **rt_def.h** — Screen/engine constants (FINEANGLES, SFRACBITS, SFRACUNIT, HEIGHTFRACTION, VIEWWIDTH, VIEWHEIGHT)
- **watcom.h** — `FixedMul()`, fixed-point arithmetic (Watcom inline assembly)
- **modexlib.h** — VGA register macros (`VGAWRITEMAP()`, `VGAMAPMASK()`, `VGAREADMAP()`), framebuffer pointers (`bufferofs`, `ylookup[]`)
- **w_wad.h** — `W_CacheLumpNum()` (resource loading)
- **z_zone.h** — Memory zone manager (for `PU_CACHE` flag)
- **rt_scale.h** — Function declarations
- **_rt_scal.h**, **rt_sc_a.h** — Assembly stubs or macro extensions
- **engine.h**, **rt_main.h**, **rt_ted.h**, **rt_vid.h**, **rt_view.h**, **rt_playr.h** — Global engine state (player, MISCVARS, lights, posts[], etc.)
