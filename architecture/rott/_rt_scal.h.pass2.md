# rott/_rt_scal.h — Enhanced Analysis

## Architectural Role

This header is a **constants definition point** for the raycasting viewport subsystem, establishing the baseline player height used throughout geometric projections and screen scaling calculations. It bridges the raw game world (260 units) into the scaled fixed-point space used by the rendering pipeline, serving as a fundamental anchor point for camera setup, wall/sprite projection, and viewport geometry calculations in the 3D renderer.

## Key Cross-References

### Incoming (who depends on this file)
- Files in the rendering subsystem that perform viewport calculations (`rt_view.c`, `rt_draw.c`, likely `rt_build.c` based on first-pass context)
- Actor/player code that needs viewport-relative positioning
- Screen projection routines that scale world geometry to screen space

### Outgoing (what this file depends on)
- **`HEIGHTFRACTION`** — a scaling factor defined elsewhere (likely in a companion header like `_rt_math.h` or `_rt_view.h`)
  - The value is unknown from provided context but establishes the fixed-point precision bits
  - Critical dependency: if HEIGHTFRACTION is 16 bits, 260<<16 gives very different precision than 260<<8

## Design Patterns & Rationale

**Fixed-Point Scaling (1990s Standard)**
- Bit-shift left by a constant instead of floating-point arithmetic
- Avoids FPU performance penalties (relevant for 486/Pentium-era target hardware)
- Allows integer-only math while maintaining sub-pixel precision
- The `260` is likely the *logical* unit height; the shift converts to *fixed-point* representation

**Single Definition Principle**
- By isolating this constant in a dedicated header, geometry changes propagate everywhere automatically
- Prevents hardcoded "260" values scattered through the codebase
- Marks it "private" (`_rt_scal`) to prevent external code from depending on the scaling representation directly

## Data Flow Through This File

1. **Input**: Raw player height (`260`) + scaling factor (`HEIGHTFRACTION`)
2. **Transformation**: Bit-shift operation produces a scaled fixed-point value
3. **Output**: `PLAYERHEIGHT` macro is consumed by:
   - Viewport setup (camera height initialization)
   - Sprite/wall projection (scaling world y-coordinates to screen space)
   - Collision detection/camera clipping boundaries
4. **Propagation**: Changes here automatically update all downstream rendering calculations without recompilation of dependent files that use the macro (macro expansion happens at compile time)

## Learning Notes

- **1990s Raycasting Pattern**: This exemplifies the era's performance-conscious design—fixed-point math was the norm for 3D software rendering before GPUs
- **Contrast to Modern Engines**: Modern engines use floats freely and let the hardware (GPU) handle precision; this shows the tight resource constraints of the mid-90s
- **Macro-Based Configuration**: Rather than inline constants or config files, critical game parameters were hardcoded macros—common in this era for zero runtime overhead
- **Architectural Clarity**: The `_rt_` prefix convention (rt = runtime; underscore = private) and the sparse 25-line file show disciplined separation of concerns
- **Fixed-Point Ubiquity**: The pattern of `value << FRACTION` appears throughout classic engines; developers studying this codebase would encounter this idiom repeatedly

## Potential Issues

- **Dependency Fragility**: If `HEIGHTFRACTION` is not defined in compilation units that include this header, silent failures or incorrect scaling could occur
- **Magic Number**: The `260` unit value has no documentation—unclear whether it's chosen for gameplay balance, level design grid dimensions, or another constraint
- **No Bounds Checking**: Nothing validates that the shift operation doesn't overflow (unlikely but possible if HEIGHTFRACTION ≥ 22 on 32-bit ints)

---

**Note**: Full architectural context (ARCHITECTURE CONTEXT) was unavailable for this analysis; inferences about caller/callee relationships are based on first-pass analysis and function names visible in the cross-reference index.
