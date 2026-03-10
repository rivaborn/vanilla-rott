# rott/rt_def.h — Enhanced Analysis

## Architectural Role

`rt_def.h` is the **foundational type and constant repository** for the entire ROTT engine. It establishes the mathematical and coordinate system primitives (angle quantization, fixed-point scaling, view dimensions) that all rendering, collision, and actor movement logic depends on. By centralizing these definitions, the engine enforces a consistent spatial model across rendering (raycast angles, view frustum), game logic (actor collision radius, map tiling), and input handling. This file effectively serves as the contract between the low-level rendering pipeline (which depends on `VIEWWIDTH`, `VIEWHEIGHT`, `ANGLES`, `MINDIST`, `FOCALLENGTH`) and higher-level game systems (which use `weapontype`, button enums, and entity flags).

## Key Cross-References

### Incoming (who depends on this file)
- **Nearly all `rott/` modules**: Given the comment "Zee big one," this header is universally included by:
  - Rendering pipeline: `rt_draw.c`, `rt_view.c` (consume `VIEWWIDTH`, `VIEWHEIGHT`, `FOCALLENGTH`, angle constants)
  - Game logic: `rt_actor.c`, `rt_playr.c`, `rt_door.c`, `rt_stat.c` (use `thingtype`, `dirtype`, flag bits)
  - Input handling: Files processing `weapontype` and button enums (attack, strafe, look, weapon selection)
  - Map/collision: `rt_map.c`, `rt_door.c` (use `MAPSPOT`, `TILEGLOBAL`, `PIXGLOBAL`, `AREANUMBER`)
- **Cross-module codebases**: `cin_*.c` (cinematics), `rt_battl.c` (battle mode), `rt_net.c` (networking) all reference shared types and flags

### Outgoing (what this file depends on)
- **`"develop.h"`**: Configuration flags (`SHAREWARE`, `SUPERROTT`, `SITELICENSE`) that conditionally define:
  - `MAXWEAPONS` (9 for shareware, 13 for full version)
  - Available weapon types (`wp_split`, `wp_kes`, `wp_bat`, `wp_dog` only in full build)
  - These branching points ensure single-source control of game balance across builds
- **`<stdio.h>`**: Minimal; likely included as precaution rather than direct use in this header

## Design Patterns & Rationale

### 1. **Fixed-Point Arithmetic Foundation**
   - `fixed` (typedef'd as `long`), `SFRACBITS=16`, `SFRACUNIT=0x10000` establish sub-tile precision
   - Rationale: 1990s CPU performance; avoids floating-point overhead on i386/486 hardware
   - Constants like `PLAYERSIZE=0x5700l`, `MINDIST=0x5800l`, `FOCALLENGTH=0x5700l` are pre-scaled to this system

### 2. **Dual Angle Systems**
   - `ANGLES` (2048) for game logic and `FINEANGLES` (2048) for rendering
   - `ANG90`, `ANG180`, etc. vs. `VANG90`, `VANG180` (separate coarse-angle set)
   - Rationale: Allows both high-precision raycasting and efficient integer angle quantization in collision/AI code

### 3. **Bit-Flag Polymorphism**
   - Flags are organized by entity category:
     - **Shared** (`FL_SHOOTABLE`, `FL_ACTIVE`, `FL_DYING`): Universal entity attributes
     - **Actor-specific** (`FL_ATTACKMODE`, `FL_AMBUSH`, `FL_STUCK`): Behavior control
     - **Player-specific** (`FL_GODMODE`, `FL_DOGMODE`, `FL_GASMASK`): Power-up and mode tracking
     - **Sprite-specific** (`FL_ROTATING`, `FL_RESPAWN`, `FL_DEADBODY`, `FL_WEAPON`): Asset/rendering hints
   - Rationale: Enables fast bit-testing without polymorphic dispatch; fits 1990s memory constraints

### 4. **Macro Abstraction Over Direct Array Access**
   - `#define MAPSPOT(x,y,plane) (mapplanes[plane][MAPSIZE*(y)+(x)])`
   - `#define AREANUMBER(x,y) (MAPSPOT((x),(y),0)-AREATILE)`
   - Rationale: Encapsulates 2D map indexing; if map layout changes, only these macros need updates

### 5. **Register Struct Accessor Macros**
   - `#define AX(r) ((r).x.eax)`, etc.
   - Rationale: DOS-era x86 interrupt/BIOS calling; abstracts platform-specific register layout (nested `x.eax` structure)

## Data Flow Through This File

1. **Engine Initialization**
   - `BuildTables()` (called early in `rt_draw.c`) uses angle constants and view dimensions to precompute sine/tangent lookup tables
   - Map parser uses `TILEGLOBAL`, `MAPSIZE`, `AREATILE` to decode tile IDs into playable area boundaries

2. **Rendering Per-Frame**
   - Raycaster loop iterates from 0 to `VIEWWIDTH` (320 pixels)
   - Each ray is quantized to `FINEANGLES` (2048-step angle system)
   - Wall heights are computed using `FOCALLENGTH` and distance scaling

3. **Game Logic Loop**
   - Actor state machines check flags (`FL_ATTACKMODE`, `FL_DYING`, `FL_STUCK`)
   - Collision tests use `PLAYERSIZE`, `MINACTORDIST`, `MINDIST` as radius/distance thresholds
   - Direction calculation uses `dirtype` enum (8-compass cardinal directions)

4. **Input & Action Processing**
   - Button enum values (0–26) drive input state machine branches
   - `weapontype` enum selects weapon-specific code paths (fire rate, projectile behavior, effects)
   - Shareware vs. full version branching happens at compile time via conditional `MAXWEAPONS`

## Learning Notes

### Engine Idioms vs. Modern Practice
- **Fixed-point everywhere**: Modern engines use floats freely; ROTT's `0x10000` scaled integers reflect 1990s CPU budgets
- **Angle quantization**: Modern engines use radians/floats; ROTT's 2048-angle system is faster to index tables and avoid trig
- **Flat flag bits**: No inheritance or component-based ECS; entities are monolithic with behavior driven by flag masking
- **Compile-time feature branching**: Full game vs. shareware split at #define time; modern engines use data-driven feature flags

### Key Insights for Engine Study
- The angle system is **central to the rendering model**: raycasts are quantized to 2048 angles, and this constrains both view resolution and collision detection precision
- **Flags are the behavior controller**: There's no virtual method table or explicit state machine; entity behavior emerges from flag checking throughout the game loop
- **Map/tile constants are baked**: `MAPSIZE=128` and tile IDs (e.g., `AREATILE=107`) suggest a fixed-size grid with hardcoded area semantics
- **Platform assumptions visible**: Register macros and `SWIFT` device constants (Cyberman joystick) show tight coupling to DOS I/O

## Potential Issues

1. **Bit-flag collision risk**: Some flags are reused across contexts (e.g., `FL_FIRSTATTACK=0x20` and `FL_ISFIRE=0x20` are both 0x20 but used in different entity types). Risk of misapplication if a developer confuses scopes.

2. **Magic number density**: Tile IDs (`AREATILE=107`, `ICONARROWS=72`, `ELEVATORTILE=72`, `PUSHABLETILE=80`) are hardcoded constants with no symbolic explanation. Fragile if tile definitions change.

3. **Dual angle systems potential confusion**: Having both `ANGLES`/`VANG*` and `FINEANGLES`/`ANG*` increases risk of off-by-one errors in angle conversions if callers mix systems.

4. **Shareware branching at compile time**: No runtime feature detection; if a full-version executable is run in an environment expecting shareware, weapon availability mismatches could cause crashes.
