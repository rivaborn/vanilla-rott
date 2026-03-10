# rott/rt_scale.c — Enhanced Analysis

## Architectural Role

This file is the **sprite rasterization subsystem** of the software-rasterized 3D engine, bridging the high-level 3D visibility determination (from `rt_draw.c`'s `ThreeDRefresh()`) and low-level pixel assembly (hardware-specific VGA column drawing). It handles perspective-correct scaling, light lookup composition, and multiplanar VGA output for all visible sprites—both world objects and HUD/weapon overlays. The file is responsible for converting the visibility list into final framebuffer pixels across multiple rendering modes (opaque, transparent, clipped, solid-color overlays).

## Key Cross-References

### Incoming (who depends on this file)
- **rt_draw.c** (`ThreeDRefresh()`) — Main 3D refresh loop iterates `vislist[]` and calls `ScaleShape()`, `ScaleTransparentShape()`, `ScaleSolidShape()` for world sprites in back-to-front order
- **rt_playr.c** — Weapon drawing; calls `ScaleWeapon()` for HUD sprite rendering
- **rt_menu.c** — UI/menu rendering; calls `DrawScreenSprite()`, `DrawScreenSizedSprite()`, `DrawNormalSprite()`
- **rt_main.c** or similar — Screen initialization/refresh loop orchestration

### Outgoing (what this file depends on)
- **rt_draw.h/rt_draw.c** — Reads `visobj_t` definition, global lighting state (`lightsource`, `fulllight`, `fog`), palette pointers (`colormap`, `redmap`, `playermaps[]`, `greenmap`), screen dimensions (`viewwidth`, `viewheight`), rendering globals (`centeryfrac`, `posts[]`, `ylookup[]`, `bufferofs`)
- **rt_def.h** — Engine constants (`FINEANGLES`, `SFRACBITS`, `SFRACUNIT`, `HEIGHTFRACTION`, `PLAYERHEIGHT`)
- **watcom.h** — Fixed-point utilities (`FixedMul()`)
- **w_wad.h** — Resource management (`W_CacheLumpNum()` with `PU_CACHE` flag for sprite data)
- **modexlib.h** — VGA mode-X register abstraction (`VGAMAPMASK()`, `VGAWRITEMAP()`, `VGAREADMAP()`, `mapmasks1/2/3` lookup tables)
- **_rt_scal.h / rt_sc_a.h** — Assembly-language column drawing primitives (`R_DrawColumn()`, `R_TransColumn()`, `R_DrawClippedColumn()`, `R_DrawSolidColumn()`)
- **rt_playr.h** — Player object (`player->angle`, `player->x`, `player->y`)
- **engine.h** — Global engine state (`MISCVARS`, `lightsource` flags, lighting state)

## Design Patterns & Rationale

### 1. **Post-Based RLE Rendering**
The sprite column data uses a custom RLE encoding (offset/length pairs, terminated by 255) rather than raw pixels. This reflects storage constraints of 1995 and allows efficient compression of tall sprites. The `ScaleMaskedPost()` and variants iterate through posts, computing screen-space Y bounds dynamically per post.

### 2. **Palette Lookup Shading**
Light levels are applied via palette indirection (`shadingtable` pointers into `colormap`, `redmap`, etc.) rather than color computation. This is characteristic of paletted VGA graphics: a 256-color palette is indexed by sprite color + light level (via `shadingtable` offset). The `SetLightLevel()` and `SetPlayerLightLevel()` functions select which palette row (0–31 shade levels in `colormap` layout) to use.

**Tradeoff**: Pre-computed palette lookups are fast (single memory dereference) but inflexible—color grading, hue shifts, or dynamic lighting require palette regeneration or multiple palette tables.

### 3. **Fixed-Point Scaling Math**
All scaling uses fixed-point arithmetic (`SFRACBITS`, `SFRACUNIT` macros, likely `1<<16` precision). The `dc_invscale` (inverse scale: screen pixels per texture sample) is computed once per sprite, then used in tight loops:
```c
topscreen = sprtopoffset + (dc_invscale * offset);
```
This avoids per-pixel division and enables perspective-correct mapping.

**Tradeoff**: Limited precision (16-bit fixed-point) vs. simplicity and speed on 1995 hardware lacking FPUs.

### 4. **VGA Planar Mode Handling**
Wide sprites require `ScaleMaskedWidePost()` / `ScaleClippedWidePost()` to render across multiple VGA memory planes (4 per byte in mode-X). The code uses lookup tables (`mapmasks1/2/3`) to determine which planes to write per pixel, then calls the underlying post function 3 times with `VGAMAPMASK()` plane selection.

**Tradeoff**: Adds complexity but necessary for VGA hardware; modern GPU renderers ignore this entirely.

### 5. **Global State for Drawing Parameters**
Rather than passing render parameters through function call stacks, the file uses file-static globals (`dc_yl`, `dc_yh`, `dc_source`, `dc_invscale`, `shadingtable`, etc.). This reduces call overhead and stack usage—critical for 1995 DOS 32-bit real-mode code.

**Tradeoff**: Makes code less modular and harder to parallelize, but enables tight assembly loops in `_rt_scal.h`.

### 6. **Occlusion via Wall Height Array**
The `posts[x].wallheight` array stores the tallest opaque wall pixel at each screen column. Sprite post rendering compares `dc_yh` against this to clip or skip columns entirely. This allows sprites to be occluded by walls without depth sorting the entire scene.

## Data Flow Through This File

```
Input:
  ThreeDRefresh() → vislist[] (sorted sprites)
         ↓
  ScaleShape(visobj_t sprite)
         ↓
  1. Load sprite patch data via W_CacheLumpNum()
  2. Compute dc_invscale (perspective scaling)
  3. Compute dc_texturemid, sprtopoffset (Y positioning)
  4. Set shadingtable based on sprite.colormap (palette selection)
         ↓
  5. Loop over sprite columns (x1 to x2):
     - Check posts[x].wallheight for occlusion
     - Call ScaleMaskedWidePost() or ScaleMaskedPost()
         ↓
  6. ScaleMaskedPost() → Loop over posts:
     - RLE decode: offset, length
     - Map texture Y to screen Y via dc_invscale
     - Set dc_yl, dc_yh (clipped screen bounds)
     - Call R_DrawColumn() (assembly) → writes to VGA framebuffer
         ↓
Output: Framebuffer pixels at (x, dc_yl..dc_yh)
```

**State mutations**:
- Global `dc_*` variables set before each post-rendering call
- `shadingtable` modified by light-level setters
- `posts[]` array read (not written) for occlusion
- VGA plane masks set via `VGAMAPMASK()` before plane-specific rendering

## Learning Notes

### 1. Column-Based Rendering (Doom-Like Architecture)
ROTT uses the Doom engine's column-rasterization paradigm: the renderer processes the framebuffer vertically (top to bottom per column) rather than horizontally. This simplifies occlusion handling and matches VGA's linear memory layout in mode-X. Modern engines use 3D APIs (OpenGL, Direct3D) with triangle rasterization and depth buffering instead.

### 2. Palette-Based Lighting
Pre-computed palette tables for each light level and object color are a key optimization. Changing a single index into the palette achieves lighting without per-pixel arithmetic. This is completely absent in modern 3D renderers, which compute lighting in shaders.

### 3. Software Rasterization with VGA Constraints
The code navigates the peculiarities of 1995 VGA hardware:
- Planar memory layout (4 pixels per byte)
- No hardware blending (translucency done via palette lookup)
- No depth buffer (wall occlusion handled manually via `posts[]`)

### 4. RLE Sprite Compression
Sprites are stored as RLE-encoded columns to save memory. Decompression happens on-the-fly during rendering, a technique absent in modern engines where textures are stored uncompressed (or compressed with hardware-friendly formats like BC1).

### 5. Fixed-Point Perspective Scaling
The scaling math using `dc_invscale` and fixed-point arithmetic predates GPU perspective correction. Modern GPUs perform this interpolation in the rasterizer automatically.

## Potential Issues

1. **Limited Validation**: Development-mode checks (`#if (DEVELOPMENT == 1)`) validate `shadingtable` range and transparency levels, but absent in release builds. No bounds checking on sprite pixel data RLE loops.

2. **Global State Coupling**: All `dc_*` and `shadingtable` globals are shared mutable state. Rendering order depends on strict sequencing; parallel sprite rendering would require per-thread state.

3. **Assembly Opacity**: The actual pixel-writing logic lives in `R_DrawColumn()` (in `_rt_scal.h` / `rt_sc_a.h`), which is not visible here. Bugs in column rendering (buffer overflows, incorrect palette lookups) are invisible to this file's review.

4. **Implicit Occlusion Semantics**: The `posts[x].wallheight` check for occlusion is implicit—there's no explicit depth comparison or early-exit guard, relying on the assembly column-drawing code to respect bounds.

5. **Translucency via Palette Lookup**: The `ScaleTransparentPost()` function selects a transparency palette (`seelevel`) via bit shifting (`((level+64)>>2)<<8`), but the actual blending math is opaque. If the palette isn't pre-computed correctly, translucency will look wrong.
