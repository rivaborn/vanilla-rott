# rott/mapsrot.h

## File Purpose
TED5 map header file defining the complete enumeration of playable levels in Rise of the Triad. Contains symbolic names for all campaign maps plus placeholder slots for future content. Serves as a centralized reference for map indices throughout the game engine.

## Core Responsibilities
- Define enum `mapnames` with all map identifiers
- Provide symbolic constants for map selection/loading
- Reserve slots (EMPTYMAP37–EMPTYMAP82) for map expansion
- Document map ordering and assignment (indices 0–83)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `mapnames` | enum | Enumeration of all in-game maps and placeholders; values 0–83 |

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This header is a compile-time constant reference. The enum values are likely used during:
- **Level loading**: `switch(current_map)` or array indexing into map data structures
- **Game initialization**: Validating map indices
- **Level selection menus**: Iterating over available (non-EMPTYMAP) entries

The sequential numbering (with gaps filled by EMPTYMAP placeholders) suggests maps are stored in a fixed-size array indexed by `mapnames` values.

## External Dependencies
- No includes or external symbols visible in this file
- Assumed to be included by game logic modules that load/manage maps (likely `rott.h`, game state manager, or level loader)

## Notes
- Map names (0–35) are campaign levels with descriptive identifiers; remainder are reserves
- `LASTMAP` (value 83) serves as a sentinel/count value
- Special characters in names (e.g., `&`, `-`, `_`) suggest these are human-readable TED5 editor names, not C identifiers, preserved as-is
