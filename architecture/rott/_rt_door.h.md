# rott/_rt_door.h

## File Purpose
Header file defining door and touch plate mechanics for the game engine. Contains structure definitions and timing/flag constants for door opening animations, push walls, and touch plate trigger systems.

## Core Responsibilities
- Define touch plate state structure (`saved_touch_type`) with timing and action tracking
- Define timing constants for door open/close animations (OPENTICS = 165 tics)
- Define flag constants for door state marking in the level
- Define push wall animation parameters (frame count and tic timing)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `saved_touch_type` | struct | Persistent state for a touch plate/trigger: action index, object reference, timing counters, and completion flags |

## Global / File-Static State
None.

## Key Functions / Methods
None—this file contains only constants and type definitions.

## Control Flow Notes
- **OPENTICS (165)**: Timer for door opening animation duration
- **Touch plate state machine**: `triggered` → `complete` → `done` flags manage touch plate lifecycle
- **tictime/ticcount**: Track animation frame timing within the plate action
- **actionindex/swapactionindex**: Support two distinct actions (e.g., open/close toggle)
- **Push wall animation**: 9 frames displayed at 3 tics per frame (AMW_TICCOUNT)
- Flags **FL_TACT** (0x4000) and **FL_TSTAT** (0x8000) likely mark door/touchplate tile properties in level data

## External Dependencies
None—self-contained header with no external includes or symbol dependencies.
