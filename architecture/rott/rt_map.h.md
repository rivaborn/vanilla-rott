# rott/rt_map.h

## File Purpose
Public interface header for map rendering and display functionality in the Rise of the Triad engine. Declares the main map display function and a cheat code handler for map visualization.

## Core Responsibilities
- Export the primary map rendering function (`DoMap`)
- Export the map cheat code handler (`CheatMap`)
- Provide a minimal public API for map-related operations from other game modules

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### DoMap
- **Signature:** `void DoMap(int x, int y);`
- **Purpose:** Render or display the game map at a specific location
- **Inputs:** `x, y` — coordinates (pixel position or tile position; not inferable from signature alone)
- **Outputs/Return:** None (void)
- **Side effects (global state, I/O, alloc):** Not inferable from header; likely modifies screen/framebuffer
- **Calls:** Not visible in header file
- **Notes:** Implementation in `rt_map.c`; coordinate semantics and rendering context undefined here

### CheatMap
- **Signature:** `void CheatMap( void );`
- **Purpose:** Activate a cheat code related to map display or traversal
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects (global state, I/O, alloc):** Not inferable from header; likely toggles map visibility or reveals fog-of-war
- **Calls:** Not visible in header file
- **Notes:** Implementation in `rt_map.c`; actual cheat behavior undefined here

## Control Flow Notes
Not inferable from header. Likely called during frame rendering (`DoMap`) and input handling / cheat code processing (`CheatMap`).

## External Dependencies
- None visible (this is a pure interface header with no includes)
- Implementation and callers defined elsewhere in the codebase
