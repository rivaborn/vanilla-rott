I'll analyze `rott/_rt_ted.h` with the cross-reference and architectural context provided. Let me create an enhanced second-pass analysis:

---

# rott/_rt_ted.h — Enhanced Analysis

## Architectural Role

This header is a **format and spatial query layer** bridging file I/O and the runtime actor/sprite system. It defines both the on-disk map file format (RLEW-compressed planes) and provides macro-based queries into the live `actorat[][]` and `sprites[][]` spatial grids that track what occupies each map tile at runtime. The conditional level filename definitions make this file a product configuration point, enabling compile-time selection of game/battle level sets across shareware, registered, and site-licensed builds.

## Key Cross-References

### Incoming (who depends on this file)
- **Level loaders / map initialization** → use `mapfiletype`, `maptype`, and RLEW format constants to read/parse `.RTL` and `.RTC` files
- **Collision/spatial query code** → use `ActorIsPushWall()`, `ActorIsWall()`, `ActorIsSpring()`, `StaticUndefined()` macros to check grid occupancy
- **Game mode setup** → use `STANDARDGAMELEVELS` / `STANDARDBATTLELEVELS` to load appropriate level files (inferred from conditional defines)

### Outgoing (what this file depends on)
- **`rt_actor.h`** → for `objtype` structure definition and actor class enums (`PWALL`, `WALL`)
- **`develop.h`** → for preprocessor flags (`SHAREWARE`, `SUPERROTT`, `SITELICENSE`)
- **Implicit globals** → reads `actorat[][]` (actor grid), `sprites[][]` (sprite grid) via macros

## Design Patterns & Rationale

**Macro-based type checking (duck typing)**: Rather than storing type tags or using virtual functions, the grid macros cast opaque pointers and inspect fields (`->which`, `->obclass`) directly. This is memory-efficient (no vtables) but unsafe—the cast assumes the pointer is valid.

**Compile-time product variants**: Build flags control level file selection at preprocessing time, avoiding runtime branching. This reflects 1990s distribution practices: the same source compiled differently for different product SKUs.

**Separation of concerns**: File format structures (`mapfiletype`, `maptype`) are distinct from runtime representation. The format is tightly coupled to RLEW compression and plane layout; the runtime uses grid-based spatial queries.

## Data Flow Through This File

**Map Loading**: Binary `RTL`/`RTC` files (read via `mapfiletype`/`maptype`) → decompressed plane data → runtime `actorat[][]` and `sprites[][]` spatial grids.

**Collision/Query**: Game loop → uses macros to poll grid for actor type at `(x, y)` → returns type classification (push wall, wall, spring, undefined static) → affects movement/interaction logic.

## Learning Notes

- **Pre-ECS spatial data**: Unlike modern ECS engines, ROTT uses dense 2D grids indexed by tile coordinates. This is fast for raycasting and spatial queries but tight on memory.
- **Hardcoded UI offsets**: Precache display positions (`PRECACHEASTRINGX`, etc.) are pixel coordinates, reflecting fixed VGA 320×200 resolution. No scalable layout system.
- **Product licensing in code**: The conditional defines show how Apogee managed multiple product tiers—same engine, different level sets and tags (`SHAREWARE_TAG`, `REGISTERED_TAG`).
- **Unsafe casts and field inspection**: The grid macros assume `actorat[][]` entries are either NULL or valid `objtype*` pointers. No tagging or discriminated unions; relies on convention.

## Potential Issues

- **Type safety**: Macros blindly cast and dereference `actorat[xx][yy]`; if the grid is corrupted or contains misaligned pointers, this will crash or misread memory.
- **Sparse grid assumptions**: Macros test `(actorat[xx][yy])` for truthiness but don't validate bounds on `xx`, `yy`. Callers must ensure indices are valid.
- **Precache layout rigidity**: UI positions are hardcoded constants; porting to different resolutions would require manual coordinate updates throughout the codebase.
