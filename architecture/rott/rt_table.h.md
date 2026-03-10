# rott/rt_table.h

## File Purpose

Defines the global state lookup table (`statetable`) that maps numeric state IDs to pointers to `statetype` definitions. This file serves as the central registry enabling O(1) runtime access to all entity behaviors, animations, and effect sequences by state ID.

## Core Responsibilities

- Provide a global indexed lookup table for game state definitions
- Initialize 660–1300 state pointers depending on game version (shareware vs full)
- Organize states by entity type: guards, enemies, NPCs, projectiles, effects, environmental
- Support hierarchical state machines by index (e.g., state 0 = s_lowgrdstand, state 1 = s_lowgrdpath4, etc.)
- Maintain symmetry between state ID and pointer position in array via careful ordering

## Key Types / Data Structures

None (pure data file). References `statetype` struct defined in `states.h`.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| statetable | `statetype * [MAXSTATES]` | global | Master state registry; indexed lookup for all game entity and effect state definitions |

## Key Functions / Methods

None. This is a static data initialization file containing no executable code.

## Control Flow Notes

**Initialization:** The array is initialized at program load time with all state pointers in a fixed order. The array index implicitly becomes the state ID used elsewhere in the codebase.

**Runtime Access:** During the main game loop and entity state transitions, code likely uses state IDs to index into `statetable[]` to fetch the current state definition (rotation, sprite, tic duration, think function, transitions).

**Conditional Compilation:** Full game (SHAREWARE=0) includes additional states for premium enemies and bosses (Darian, Heinrich, Dark Monk, NME boss, etc.); shareware version excludes these.

## External Dependencies

- **Include:** `states.h`
  - Defines `statetype` struct (rotate, shapenum, tictime, think function pointer, condition, next)
  - Defines `MAXSTATES` constant (1300 or 660)
  - Declares all individual state objects (e.g., `extern statetype s_lowgrdstand`)
  
- **External state symbols used but defined elsewhere:**  
  All `s_*` identifiers (e.g., `s_lowgrdstand`, `s_explosion1`, `s_player`, `s_darkmonkstand`) are declared in `states.h` and implemented in other `.c` files. This file only collects pointers to them.
