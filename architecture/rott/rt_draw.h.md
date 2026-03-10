# rott/rt_draw.h

## File Purpose
Header file declaring the core drawing and rendering system for the ROTT 3D engine. Defines data structures for visible objects, ray-tracing variables, camera state, and provides entry points for the main 3D refresh loop, screen effects, and cinematic sequences.

## Core Responsibilities
- Define the `visobj_t` structure and visible object management (`vislist`, visibility pointers)
- Declare ray-tracing state variables (`xintercept`, `yintercept`, `mapseen`)
- Manage camera transformation state (`viewx`, `viewy`, `viewangle`, sin/cos/tan tables)
- Provide light source management via the `lights` array and macros (`LightSourceAt`, `SetLight`)
- Declare the main 3D rendering function (`ThreeDRefresh`) and frame-time logic (`CalcTics`)
- Declare screen/UI functions (title screens, cinematics, credits, screen saver)
- Expose display control functions (`FlipPage`, `TurnShakeOff`)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `visobj_t` | struct | Stores rendering parameters for a visible object (position, shape, texture, height, colormap) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `whereami` | int | global | Current position/location identifier |
| `shadingtable` | byte * | global | Lookup table for shading calculations in `DrawPost` |
| `tilemap` | word [MAPSIZE][MAPSIZE] | global | Wall tile data for the game map |
| `spotvis` | byte [MAPSIZE][MAPSIZE] | global | Per-tile visibility state |
| `tantable` | short [FINEANGLES] | global | Tangent lookup table |
| `sintable` | fixed [FINEANGLES+FINEANGLEQUAD+1] | global | Sine lookup table |
| `costable` | fixed * | global | Cosine lookup table (pointer; shares memory with `sintable`) |
| `viewx`, `viewy` | fixed | global | Camera focal point in world space |
| `viewangle` | int | global | Camera viewing direction |
| `viewsin`, `viewcos` | fixed | global | Precomputed sin/cos of `viewangle` |
| `vislist` | visobj_t [MAXVISIBLE] | global | Array of visible objects for current frame |
| `visptr`, `visstep`, `farthest` | visobj_t * | global | Pointers for iterating and managing visible object list |
| `xintercept`, `yintercept` | long | global | Ray-wall intersection coordinates |
| `mapseen` | byte [MAPSIZE][MAPSIZE] | global | Map areas explored/seen by player |
| `lights` | unsigned long * | global | Light source intensity grid |
| `tics`, `wstart`, `fandc` | int | global | Frame timing counters |
| `hp_startfrac`, `hp_srcstep` | int | global | Height-plane rasterization parameters |
| `levelheight`, `maxheight`, `nominalheight` | int | global | Level and object height constraints |
| `actortime`, `drawtime` | int | global | Performance timing counters |
| `c_startx`, `c_starty` | int | global | Calculation/camera start positions |
| `dirangle8`, `dirangle16` | int [9], [16] | global | Direction angle tables (8 and 16-way) |
| `firstcoloffset` | int | global | Screen column offset for rendering start |

## Key Functions / Methods

### ThreeDRefresh
- Signature: `void ThreeDRefresh(void)`
- Purpose: Main 3D rendering loop; performs ray tracing, visible object sorting, and screen updates each frame
- Inputs: None (uses global view state and tilemap)
- Outputs/Return: None
- Side effects: Updates `vislist`, screen buffer, modifies global render state
- Calls: (not visible in header; engine core function)
- Notes: Primary entry point for frame rendering; called once per game tick

### BuildTables
- Signature: `void BuildTables(void)`
- Purpose: Initialize sin/cos/tan lookup tables at startup
- Inputs: None
- Outputs/Return: None
- Side effects: Populates `sintable`, `costable`, `tantable`
- Calls: (not visible in header)
- Notes: Called during engine initialization

### CalcHeight
- Signature: `int CalcHeight(void)`
- Purpose: Calculate object or sprite height based on distance and view parameters
- Inputs: None (uses global view state)
- Outputs/Return: Calculated height in pixels
- Side effects: None
- Calls: (not visible in header)
- Notes: Used for sprite scaling during rendering

### FlipPage
- Signature: `void FlipPage(void)`
- Purpose: Swap display buffer; show rendered frame to screen
- Inputs: None
- Outputs/Return: None
- Side effects: Updates display hardware/framebuffer
- Calls: (not visible in header)
- Notes: Called at end of frame render cycle

### RotateBuffer
- Signature: `void RotateBuffer(int startangle, int endangle, int startscale, int endscale, int time)`
- Purpose: Apply rotation and scaling transformation to the frame buffer over time
- Inputs: `startangle`, `endangle` (rotation range), `startscale`, `endscale` (scaling range), `time` (duration)
- Outputs/Return: None
- Side effects: Modifies framebuffer; used for wipe/transition effects
- Calls: (not visible in header)
- Notes: Likely used for cinematic transitions and visual effects

## Screen / UI Functions
The following are declared but represent higher-level game logic (not core rendering):
- `ApogeeTitle()`, `DopefishTitle()`: Title screen sequences
- `DoEndCinematic()`: End-game cinematic
- `DoCreditScreen()`: Credits sequence
- `DoLoadGameSequence()`: Load game UI
- `DoInBetweenCinematic(int yoffset, int lump, int delay, char *string)`: Mid-level cinematics
- `DoMicroStoryScreen()`: Story screens
- `StartupScreenSaver()`, `ShutdownScreenSaver()`, `UpdateScreenSaver()`: Screen saver management
- `RotationFun()`: Demo/rotation effect
- `GetRainBoundingBox()`: Rain effect bounds
- `AdaptDetail()`: Adjust detail level (performance tuning)
- `CalcTics()`: Calculate frame timing
- `TurnShakeOff()`: Disable screen shake

## Control Flow Notes
This header defines the **refresh/render loop** interface. `ThreeDRefresh()` is the frame-synchronous entry point; it uses global camera state (`viewx`, `viewy`, `viewangle`) and the tilemap to perform ray tracing, populate `vislist`, and update the screen. Math tables are built once at startup (`BuildTables`), and display is swapped at frame boundary (`FlipPage`). Light and visibility caches (`lights`, `mapseen`, `spotvis`) persist across frames. High-level UI (title screens, cinematics) sit above this core rendering system.

## External Dependencies
- Standard C types and constants (e.g., `MAPSIZE`, `FINEANGLES`, `FINEANGLEQUAD`, `MAXVISIBLE` — defined elsewhere)
- Game engine fundamental types: `fixed` (fixed-point), `byte`, `word`
- Likely includes from a main header (rt_main.h or equivalent) for type definitions and map constants
