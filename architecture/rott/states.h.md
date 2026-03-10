# rott/states.h

## File Purpose
Declares the core state machine infrastructure for the game engine, including the `statetype` structure that represents finite states for all game entities (enemies, NPCs, effects, projectiles, player). Provides a global state table and extern declarations for hundreds of specific state instances used throughout gameplay.

## Core Responsibilities
- Defines `statetype` structure for state machine nodes (sprite, timing, think function, condition)
- Declares global `statetable` array indexing all available states
- Provides state behavior flags (`SF_*`) for conditional state logic
- Declares extern state objects for all enemy types, effects, and environmental hazards
- Segregates shareware vs. full version state counts via conditional compilation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `statetype` | struct | Game entity state node: holds animation frame, duration, AI think function, and next state pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `statetable` | `statetype*[]` | global | Array of up to 1300 (full) or 660 (shareware) state pointers for fast lookup |
| `s_lowgrdstand`, `s_lowgrdchase1`, etc. | `statetype` | global | Hundreds of extern state objects for specific entity behaviors |

## Key Functions / Methods
None. This file is purely a declaration header with macro definitions and extern state object declarations.

## Macros & Flags

**State Flags** (bitfield modifiers):
- `SF_CLOSE` (0x01) – Door/trap closing behavior
- `SF_CRUSH` (0x02) – Crushing hazard active
- `SF_UP` / `SF_DOWN` (0x04 / 0x08) – Directional movement
- `SF_SOUND` (0x10) – Play associated sound
- `SF_BLOCK` (0x20) – Blocks movement
- `SF_EYE1/2/3` (0/1/2) – Eye state variants
- `SF_DOGSTATE` (0x40) – Dog-specific behavior
- `SF_BAT` / `SF_FAKING` / `SF_DEAD` (0x80) – Bat, faking, death states

**Build Configuration**:
- `MAXSTATES`: 1300 (full game) or 660 (shareware)
- Includes `develop.h` for debug/platform flags

## Control Flow Notes
States are part of a frame-based state machine. Each entity holds a current state pointer; the engine iterates states calling their `think()` function on each frame, then transitions to `next` state based on conditions. State durations (`tictime`) control frame transitions. The `shapenum` field references sprite/animation frames; -1 means read from entity's temporary storage.

## External Dependencies
- **Local include**: `develop.h` (debug config, `SHAREWARE` flag)
- **Defined elsewhere**: All `statetype` struct definitions; `think` function implementations for each state's AI logic

## Notes
- **Naming convention**: States follow pattern `s_[entitytype][action]` (e.g., `s_blitzchase1`, `s_enforcershoot3`, `s_explosion1`)
- **Entity categories**: Low/high guards, strikers, blitz (boss), enforcers, robo-guards, special enemies (dark monk, NME machine), environmental hazards, effects, projectiles, player states
- **No implementation**: This is a declaration-only header; state structures are defined in corresponding `.c` files
- **Linked list structure**: The `next` pointer chains states together, forming sequences for multi-frame animations or behavior chains
