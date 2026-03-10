# rott/lumpy.h — Enhanced Analysis

## Architectural Role

This header defines the core **resource format contracts** between the asset loading pipeline and the rendering/drawing subsystems. Every sprite, font, and bitmap in ROTT passes through these type definitions—making `lumpy.h` a critical hub in the resource-to-screen data flow. The file bridges binary asset deserialization (resource loaders) and consumption (rendering, font drawing, sprite animation).

## Key Cross-References

### Incoming (who depends on this file)
- **Rendering subsystems**: `rt_draw.c` (CalcHeight, CalcRotate) and `rt_view.c` consume patch_t/lpic_t for sprite drawing
- **Font/text rendering**: `rt_text.c` (CacheLayoutGraphics) uses font_t and cfont_t for character rasterization
- **Static/sprite animation**: `rt_stat.c`, `rt_stat.h` (AnimateWalls, AddAnimStatic, AddStatic) use patch_t and transpatch_t for animated sprites and wall effects
- **Resource loaders**: Implicit—any code deserializing `.pic`, `.lbm`, `.fnt` assets must populate these structures
- **Cinematic system**: `cin_*.c` files likely use lbm_t for cinematic graphics

### Outgoing (what this file depends on)
- **Standard C types only**: No dependencies on other ROTT headers—this file stands alone as a type definition layer
- Implicitly read by: rendering code, font rasterizers, collision/animation systems

## Design Patterns & Rationale

1. **Flexible Array Pattern**: `data` field at end of struct enables single-allocation buffer (header + variable-length pixel/raster data). This is essential for efficient asset loading—deserialize directly into one `malloc()`-ed block.

2. **Column Offset Optimization** (patch_t/transpatch_t): The `collumnofs[320]` array is a column-by-column lookup table for **masked sprite rendering**—a 1990s optimization allowing fast skip of transparent pixels during drawing. Only `[0..width-1]` are valid; `[0]` points to `&collumnofs[width]` (i.e., offset of first pixel data). This supports *post-header offset encoding*—no need to load offsets from disk.

3. **Self-Contained Resources**: cfont_t includes inline `pal[0x300]` (768-byte RGB palette), meaning color fonts are fully self-describing and can be swapped without global palette state.

4. **Size Variants**: pic_t (8-bit dimensions) vs. lpic_t (16-bit dimensions + origin offsets) vs. patch_t (column-based) reflect different use cases—simple fullscreen graphics, large backgrounds, and masked sprites.

## Data Flow Through This File

```
Resource File (.pic, .lbm, .fnt)
         ↓ (deserialization)
    [lumpy.h type definition]
         ↓ (populated in RAM)
  Asset manager / cache
         ↓
  [Rendering pass]
     ├→ rt_draw.c: reads pic_t/lpic_t/patch_t, renders to framebuffer
     ├→ rt_text.c: reads font_t/cfont_t, renders text
     └→ rt_stat.c: reads patch_t/transpatch_t for animated statics
```

Each structure is a **static type contract**—once loaded, these types never change at runtime; only the `data` payload is interpreted differently (as pixel/raster bytes).

## Learning Notes

- **Why "lumpy"?** Likely refers to LUMP-based asset packaging (common in DOS games, famously used in Doom). These are the in-RAM representations of lumped graphics.

- **Era-specific idioms**: 
  - Fixed-size column offset arrays (collumnofs[320]) reflect 320-pixel-wide video modes (standard VGA)
  - byte/short mix reflects memory scarcity (8-bit where possible, 16-bit where needed)
  - No padding/alignment annotations—assumes tightly-packed binary layout

- **Modern contrast**: 
  - Today engines use asset pipelines with runtime-agnostic formats (e.g., glTF, PNG) and lazy-loaded resource systems
  - ROTT's approach is a **tight asset-to-memory-to-screen pipeline**, characteristic of DOS/early-90s real-time constraints

- **Architectural insight**: This file is the **contract layer**—it doesn't implement loading or rendering; it *defines* the boundary. Well-chosen because it isolates asset format knowledge from rendering logic.

## Potential Issues

- **collumnofs[320] size mismatch**: Structures allocate 320 entries, but only `[0..width-1]` are valid per the comment. If rendering code doesn't check bounds or validate width, off-by-one or out-of-bounds access is possible.
- **Implicit memory layout**: Packed structures assume no compiler padding. If compiled with non-default alignment, binary asset deserialization will corrupt data.
- **No version or magic field**: Resource files have no format identifier—version mismatch between asset builder and engine would silently corrupt rendering.
