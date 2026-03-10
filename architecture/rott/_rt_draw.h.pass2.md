# rott/_rt_draw.h — Enhanced Analysis

## Architectural Role
This private header defines the internal interface for the **rendering backend**, specifically the 3D projection and sprite drawing layer that bridges game objects and screen output. It's positioned between the high-level game logic (actors, weapons, lighting) and low-level framebuffer manipulation. The file establishes the contract that `rt_draw.c` must implement, encapsulating the perspective transformation and lighting calculation required for the engine's software rasterizer.

## Key Cross-References

### Incoming (who depends on this file)
- **rt_draw.c**: Main implementation file for all declared functions
- **Game loop / Frame render**: `DrawScaleds()` is called during render phase to draw all dynamic objects
- **Weapon system**: `DrawPlayerWeapon()` renders HUD weapon sprite during frame update
- **Lighting system**: `SetSpriteLightLevel()` is called per-sprite to apply position-dependent lighting (likely from `rt_light.c` or `rt_stat.c`)

### Outgoing (what this file depends on)
- **develop.h**: Build configuration (SHAREWARE flag controls weapon graphics count)
- **Game object types**: `objtype` (from game actor system), `visobj_t` (visibility/rendering wrapper)
- **Screen/framebuffer**: Implicit dependency; functions modify display state

## Design Patterns & Rationale

**Underscore-prefix convention**: `_rt_draw.h` signals this is private/implementation-detail header, not part of public rendering API. Forces consumers to go through `rt_draw.h` instead.

**Separation of concerns**:
- Geometric transforms (`TransformPlane`, `CalcRotate`) isolated from visibility (`DrawScaleds`, `SetSpriteLightLevel`)
- Lighting calculated separately (`SetSpriteLightLevel`) rather than baked into sprite draw, allowing per-frame dynamic updates

**Fixed-point math**: Constants like `MINZ (0x2700)`, `GOLOWER (0x38000)` suggest fixed-point 16.16 or similar, traded performance for precision (no floating-point overhead on 486-era hardware).

**Conditional compilation**: Shareware/registered build difference baked into header via macro—cleaner than runtime flags.

## Data Flow Through This File

1. **Per-frame render phase**:
   - Main loop calls `DrawScaleds()` → iterates all visible objects
   - For each object: `CalcRotate()` determines animation frame/facing
   - `TransformPlane()` projects 3D coords to 2D screen space, applies Z-culling (MINZ check)
   - `SetSpriteLightLevel()` samples world lighting at object position, sets sprite color/brightness
   - `DrawPlayerWeapon()` rendered last as HUD overlay

2. **State management**: `screensaver_t` tracks animated menu background—persistent position/rotation/scale, updated each frame via delta fields (`dx`, `dy`, `dangle`, `dscale`).

3. **Constants define pipeline bounds**:
   - `MAXVISIBLEDOORS (30)`: Frustum culling threshold
   - `MAXDRAWNTICS (40)`: Frame skip/dropout threshold
   - `MINZ (0x2700)`: Z-buffer near plane

## Learning Notes

**1990s software rasterizer patterns**: 
- No GPU; per-pixel lighting calculated in C, suggesting tight inner loops
- Fixed-point math pervasive (see `0x2700` constants)
- Visibility culling manual (MAXVISIBLEDOORS)

**Idiomatic to this engine**:
- `visobj_t` wrapping suggests renderer works with an intermediate visibility list, not raw game objects—common optimization to decouple logic from rendering
- Plane transformation suggests grid-based or portal-based world geometry (Wolfenstein 3D style), not full 3D meshes

**Modern contrast**: Today's engines use GPU instancing + compute shaders for this phase; ROTT does it in CPU-bound C loops.

## Potential Issues

None clearly inferable from header alone. However:
- **Coupling risk**: `screensaver_t` struct is specific to menu code but lives in render header—suggests possible tight coupling between menu and render subsystems
- **Z-plane assumptions**: `MINZ` constant suggests assumptions about coordinate space scaling; data flowing from `TransformPlane` must match these bounds or clipping may fail silently
