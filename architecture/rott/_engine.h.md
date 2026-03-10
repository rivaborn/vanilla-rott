# rott/_engine.h

## File Purpose
Private header file that defines internal utility macros for the rendering engine. Contains conditional compilation guards and helper macros for spatial comparisons and sign calculation.

## Core Responsibilities
- Define private engine-internal macros
- Provide tile comparison logic for wall rendering or spatial checks
- Provide sign function for directional calculations

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `posts` | array of structures with `posttype` and `offset` members | global | Rendering engine's wall segment/column data |

## Key Functions / Methods
None (macro-only file).

## Macros

### NOTSAMETILE(x1, x2)
- Purpose: Determine if two tile indices reference different rendering primitives
- Logic: Compares both `posttype` and `offset` fields; returns true if either differs
- Usage context: Likely used in raycasting or BSP traversal to detect wall transitions

### SGN(x)
- Purpose: Return the sign of a value as +1 or -1
- Logic: Ternary returning 1 if positive, -1 otherwise (0 treated as -1)
- Notes: Does not handle zero specially; not a true sign function for mathematical use

## Control Flow Notes
Not applicable; this is a utility header with no flow.

## External Dependencies
- `posts` (defined elsewhere): global rendering primitive array
- No #include directives; intended for inclusion in engine implementation files
