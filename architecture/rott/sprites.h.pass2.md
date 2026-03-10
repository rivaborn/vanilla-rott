# rott/sprites.h — Enhanced Analysis

## Architectural Role

This file is the **sprite ID registry** that bridges game logic, animation, and rendering systems. It serves as the single source of truth for naming sprite frames, enabling actor AI (`rt_actor.c`, `rt_playr.c`), the animation system (`rt_stat.c`), and the rendering pipeline (`rt_draw.c`) to reference animations by symbolic names rather than magic numbers. The enum maintains order to match sprite lump resource indexing—a critical invariant for the runtime sprite loader.

## Key Cross-References

### Incoming (who depends on this file)
- **Actor AI subsystem** (`rt_actor.c`, `rt_playr.c`, `_rt_acto.h`): Uses sprite IDs to set animation sequences for player states (movement, shooting, pain, death) and enemy behavior
- **Animation/state system** (`rt_stat.c`, `_rt_stat.h`): References hazard sprites (blades, fire, crushers) and gib sprites for dynamic world objects
- **Rendering pipeline** (`rt_draw.c`): Looks up sprite frames by ID to draw actors and effects on screen
- **Effect spawning** (cinematic, explosion handlers): Uses sprite IDs like `SPR_EXPLOSION*`, `VAPORIZED*` to trigger visual effects
- **Collision/physics** (`rt_actor.c`): May reference sprite dimensions implicitly through animation state changes
- **Weapon system**: References `W_*` prefixed sprites for shareware-gated weapon animations (e.g., `W_BJMISS*` for Billy Joe missile frames)

### Outgoing (what this file depends on)
- **develop.h**: Conditional compilation flags (`SHAREWARE`, `SHAREWARE == 0`) gate full-version content—sprites prefixed `W_` are excluded from shareware builds
- **Sprite resource loader (implicit)**: Assumes a sprite lump manager elsewhere that maps enum indices to image data via array indexing

## Design Patterns & Rationale

### 1. **Enum-as-Registry Pattern**
The entire sprite ID set is a single flat enum (`actornames_t`). This 1990s design pattern:
- Avoids pointer tables and indirection (cheap on memory/performance)
- Guarantees sequential IDs matching sprite resource order
- Makes ID→sprite lookup trivial: `sprite_data[enum_id]`
- **Tradeoff**: No spatial locality or semantic grouping; adding/removing sprites requires careful index management

### 2. **Naming Convention as Documentation**
Sprite names encode animation semantics:
- **Prefix**: `SPR_` (standard), `W_` (weapon, shareware-gated)
- **Actor ID**: `LOWGRD`, `HIGHGRD`, `STRIKE`, `BLITZ`, `ENFORCER`, `ROBOGRD`
- **Action**: `SHOOT`, `S` (standing), `W` (walking), `PAIN`, `DIE`, `WDIE` (water death)
- **Direction/Frame**: `W11` = walk direction 1, frame 1; consistent 4-direction × 8-frame pattern

This eliminates need for a separate animation definition table—developers can infer frame sequences by name.

### 3. **Conditional Compilation for Build Variants**
The `W_*` prefixed sprites and conditional blocks (`#if SHAREWARE == 0`) allow single-source maintenance of shareware vs. full versions without code duplication. The shareware build excludes premium weapon frames, enforcer animations, and advanced effects.

### 4. **Commented-Out Sprites as Feature Flags**
Extensive commented code (e.g., `SPR_LOWGRD_USE*`, `SPR_ENFORCER_USE*`) indicates removed features or placeholder animations. Rather than delete, they're left for future revival or historical reference—a practical approach for shipped titles.

## Data Flow Through This File

```
Sprite Enum ID (e.g., SPR_LOWGRD_W11)
    ↓
[Included by actor/rendering subsystems]
    ↓
Runtime sprite index lookup: spritedata[SPR_LOWGRD_W11]
    ↓
Returns sprite frame (image data, dimensions, timing)
    ↓
Renderer draws frame; animation system advances ID each tick
```

**Critical invariant**: Enum declaration order **must** match sprite lump file order. If misaligned, sprite graphics render as wrong actor animations. No runtime validation catches this—it's a link-time (lump-packing) contract.

## Learning Notes

### Idiomatic to This Engine/Era
- **No sprite inheritance**: Unlike modern ECS, each actor hardcodes its sprite sequence names. Shared animations (pain/death) must be manually copied or aliased.
- **Manual animation state machines**: Actor code explicitly sets `actor.sprite = SPR_LOWGRD_PAIN1` and increments the index per frame. Modern engines use animation state graphs or blending.
- **No spatial sorting hints**: Sprite names don't encode depth, layer, or sorting priority—that's resolved at render time.
- **Asset packing coupling**: The enum order is tightly coupled to the sprite lump build process (not shown here). Adding a sprite requires edits in *two* places: this enum and the resource file.

### Connections to Game Engine Concepts
- **State encoding via enum**: Actor behavior is partially encoded in sprite selection; a developer reading `current_sprite` can infer actor state without inspecting an AI machine.
- **Direct indexing (pre-scene graph)**: No actor → sprite ID → resource table indirection; sprites are a global namespace indexed by integer. Efficient but inflexible.
- **Animation frame-by-frame**: This is frame-based animation, not skeletal or morph-based; each pose is a discrete sprite frame.

## Potential Issues

1. **Fragility of enum order**: If sprites are added midway through the enum (not at the end), all subsequent sprite IDs shift, breaking old save files or replays that serialize sprite IDs. The codebase doesn't appear to have save versioning for this.

2. **No sprite ID validation**: Code can reference undefined or out-of-range IDs (e.g., `SPR_LOWGRD_W99`). A bounds check or assertion in the renderer would catch this.

3. **Commented-out sprites waste enum space**: IDs like `UBLADE10` and `CRUSHDOWN9` are reserved but unused, taking up positions that could be reused. Over time, this inflates the enum.

4. **Shareware coupling**: The `W_*` prefix is brittle; it relies on string matching or manual review. A `#ifdef` guard or a separate enum would be clearer.

5. **No metadata**: Sprites lack intrinsic properties (hitbox, damage, frame duration, sound trigger). These are hardcoded in actor logic, making sprite reuse difficult.
