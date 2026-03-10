# rott/engine.h

## File Purpose
Public interface header for the rendering engine module. Declares the main `Refresh()` rendering function, exports the wall-rendering data structure (`wallcast_t`) and related state, and provides a utility macro for querying the map grid.

## Core Responsibilities
- Declare the primary frame-rendering entry point (`Refresh`)
- Export the wall-casting result array (`posts`) used to draw vertical wall segments
- Track the last rendered camera position (`lasttilex`, `lasttiley`)
- Provide map-grid utility macro (`IsWindow`)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| wallcast_t | struct | Encodes a single rendered wall column: offset, height, clipping bounds, texture ID, lump, post type, alternate tile |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| posts | wallcast_t[321] | global | Array of wall-rendering data, one per screen column (max 321 columns) |
| lasttilex | int | global | Cached camera X position (detect if recalculation needed) |
| lasttiley | int | global | Cached camera Y position (detect if recalculation needed) |

## Key Functions / Methods

### Refresh
- **Signature:** `void Refresh(void)`
- **Purpose:** Main rendering function; updates the display with the current game frame
- **Inputs:** None (reads global state)
- **Outputs/Return:** None (void)
- **Side effects:** Updates screen/framebuffer; processes `posts` array and camera state
- **Calls:** Not visible from this file (implementation in engine.c)
- **Notes:** Called once per frame in the main game loop; the primary engine-facing entry point for rendering

## Control Flow Notes
Fits into the **frame/render** phase of the game loop. `Refresh()` is called each frame to redraw the 3D view. The `posts` array holds pre-computed wall-rendering data (populated by ray-casting or wall-traversal logic), which `Refresh()` then draws to the screen. The `lasttilex`/`lasttiley` globals likely cache the camera position to skip redundant calculations.

## External Dependencies
- `MAPSPOT` macro (used in `IsWindow()`): defined elsewhere (likely a map/tile header)
- `Refresh()` implementation: engine.c
