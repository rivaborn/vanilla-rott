# rott/rt_def.h

## File Purpose
Global constant and type definitions for the ROTT engine. Defines fundamental mathematical constants (angles, distances), screen/view dimensions, game entity types, input controls, weapon types, and flag bits for actors, sprites, and game state. Central foundation included by most engine modules.

## Core Responsibilities
- Define engine-wide constants: view dimensions (320×200), angle system (2048 angles), tile/pixel scaling units
- Define fundamental data types used throughout the codebase (`byte`, `word`, `fixed`, `boolean`)
- Enumerate game entity types (actor, sprite, wall, door)
- Enumerate input controls (attack, strafe, look, weapons, movement)
- Enumerate weapon types (pistol, MP40, bazooka, special weapons)
- Define flag bits for entity attributes (shootable, active, dying, etc.)
- Define map constants (tile size, map dimensions, tile IDs)
- Provide macro helpers for register access, map indexing, and area calculations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `byte` | typedef | Unsigned 8-bit value |
| `word` | typedef | Unsigned 16-bit value |
| `longword` | typedef | Unsigned 32-bit value |
| `fixed` | typedef | Fixed-point arithmetic (long) |
| `boolean` | enum | Boolean type (false=0, true=1) |
| `dirtype` | enum | 8 cardinal directions + nodir |
| `thingtype` | enum | Entity category: SPRITE, WALL, ACTOR, DOOR, PWALL, MWALL |
| `weapontype` | enum | Weapon selection: pistols, MP40, bazooka, heatseeker, special weapons (conditional on SHAREWARE) |
| button enum (unnamed) | enum | Input button codes: attack, strafe, look, weapon select, etc. (26+ button codes) |

## Global / File-Static State
None.

## Key Functions / Methods
None. (Pure definitions header.)

## Control Flow Notes
This header is foundational and likely included early in compilation order. Constants are used throughout engine initialization, game loop, and rendering:
- **Angles**: `FINEANGLES` and `ANGLES` drive rotation calculations and raycast angle indexing
- **View constants**: `VIEWWIDTH`, `VIEWHEIGHT`, `MAXSCANLINES` define viewport dimensions for rendering
- **Map/tile constants**: `MAPSIZE`, `TILEGLOBAL`, `PIXGLOBAL` used during map parsing and actor placement
- **Flags**: Flag bits checked during entity updates, collision detection, and rendering decisions
- **Weapons/buttons**: `weapontype` and button enums drive input handling and player state machines

## External Dependencies
- **`<stdio.h>`** – Standard I/O (likely used elsewhere, not directly in this file)
- **`"develop.h"`** – Development configuration flags (SHAREWARE, SUPERROTT, SITELICENSE; controls conditional weapon/game-mode compilation)

---

### Notes
- **Fixed-point math**: `SFRACBITS`, `SFRACUNIT`, `FRACUNIT` indicate 16-bit fractional scaling for sub-tile precision
- **Angle system**: 2048 fine angles divided into quads; `ANG90`, `ANG180`, etc. provide cardinal angle constants for both fine and coarse angle systems
- **Weapon branching**: Shareware build has 9 weapons; full version adds 4 more (split, kes, bat, dog)
- **Macro helpers**: `MAPSPOT()`, `AREANUMBER()` abstract map access; `AX/BX/CX/DX/SI/DI` macros suggest DOS-era x86 register structs
- **Device support**: SWIFT constants suggest input device abstraction (Cyberman support)
