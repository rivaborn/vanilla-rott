# rott/_rt_vid.h — Enhanced Analysis

## Architectural Role

This private header provides a **coordinate-notation adapter layer** for the video rendering subsystem. It sits between high-level rendering code (in `rt_vid.c`) and a lower-level video library (`VL_*` functions), converting endpoint notation (intuitive for callers) to start+length notation (efficient for the underlying graphics API). The design reflects 1990s game engine architecture where thin wrapper macros reduce boilerplate and enforce consistent coordinate calculations across the codebase.

## Key Cross-References

### Incoming (who depends on this file)
- **`rt_vid.c`** — The only intended client; includes this header and uses all four line-drawing macros in its rendering pipeline
- Likely called during frame rendering phases where UI elements, map overlays, or debug visualization needs simple geometric primitives

### Outgoing (what this file depends on)
- **`VL_Hlin`, `VL_Vlin`** — Low-level solid-color line drawing; presumably in a video library module (possibly `rt_vid.c` itself or another low-level video support file)
- **`VL_THlin`, `VL_TVlin`** — Textured variants; suggest the lower layer supports both solid fills and texture-mapped line drawing
- No explicit includes; assumes callers have already included necessary headers

*Note: The cross-reference index provided does not list VL_* or VW_* macros, suggesting these are internal/private APIs not exposed outside the video subsystem.*

## Design Patterns & Rationale

**Macro-based Coordinate Conversion Adapter:**
- Converts intuitive endpoint ranges (e.g., `x=10, z=20`) to start + length (e.g., `10, 11`) via the `+1` adjustment
- Avoids scattered arithmetic logic; all callers get consistent, readable expressions
- Why macros not functions: Zero runtime overhead; inline expansion preserves call-site performance
- Why endpoint notation: More natural for drawing between two pixel coordinates; less error-prone than mental math on lengths

**Textured vs. Solid Variants:**
- Parallel APIs (VW_Hlin/VW_THlin, VW_Vlin/VW_TVlin) suggest the engine supports both modes
- Textured versions take an `up` (update?) flag instead of color, implying a different rendering state or palette-cycling behavior

## Data Flow Through This File

1. **Input**: Caller provides endpoint coordinates (x, z for horizontal; y, z for vertical) plus rendering parameters (color or update flag)
2. **Transform**: Macros compute `length = endpoint - startpoint + 1` for inclusive ranges
3. **Output**: Expanded call to VL_* functions with start coordinate and computed length

Example: `VW_Hlin(10, 20, 5, RED)` → `VL_Hlin(10, 5, 11, RED)` (horizontal line from x=10 to x=20 at y=5 with length 11)

## Learning Notes

- **Layer separation**: Reflects classic 90s DOS/VESA architecture where low-level graphics code works in chunks/lengths for DMA and register efficiency
- **Idiom**: Using macros as lightweight adapters was common before C++ inlining; keeps performance-critical code fast without function call overhead
- **PIXTOBLOCK constant** (=4) is defined but unused here; likely represents a pixel-to-block conversion factor for the underlying graphics mode (e.g., 320÷4=80 blocks)
- **Modern equivalent**: Today's engines would use inline functions or constexpr, or push this logic into the caller; the wrapper here adds slight friction for code clarity

## Potential Issues

- **Off-by-one risk**: The `+1` in length calculations is correct for inclusive endpoints, but if a caller forgets this semantics when using VL_* directly, they'll draw one pixel too few/many
- **Unused constant**: `PIXTOBLOCK` is defined here but no usage is visible; either it's used only in `.c` files or it's vestigial
- **Incomplete API?**: Only horizontal and vertical lines are wrapped; diagonal lines, circles, or filled primitives would need separate macros or different functions
