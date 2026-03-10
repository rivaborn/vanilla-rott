# rott/cin_def.h

## File Purpose
Header file defining data structures and constants for the cinematic sequence system. Provides linked-list based event scheduling, effect types (backgrounds, sprites, palettes, FLI animations), and fixed-point math definitions for smooth animation interpolation.

## Core Responsibilities
- Define cinematic event enumeration (backgrounds, sprites, backdrops, palettes, fade, FLI, etc.)
- Define linked-list node types for events and actors in the cinematic timeline
- Define effect-specific parameter structures (sprites, backdrops, FLI animations, palettes)
- Provide fixed-point arithmetic constants for sub-pixel-precision transformations
- Define capacity limits for cinematic events and actors

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `enum_eventtype` | enum | Categorizes cinematic effects (backgrounds, sprites, fadeout, etc.) |
| `eventtype` | struct | Linked-list node; ties a time offset, effect type, and effect data |
| `actortype` | struct | Linked-list node; actor/effect in the cinematic with type and data pointer |
| `flicevent` | struct | FLI animation file reference (name, loop, usefile flags) |
| `spriteevent` | struct | Sprite animation parameters (position, scale, frame info, velocity) |
| `backevent` | struct | Backdrop/background scrolling (width, offset, velocity, pixel data) |
| `paletteevent` | struct | Palette effect reference (name only) |

## Global / File-Static State
None.

## Key Functions / Methods
None (header file with type definitions only).

## Control Flow Notes
The cinematic system likely uses `eventtype` as a timeline queue, with each node holding a time offset and a pointer to effect-specific data (typed via `enum_eventtype`). The `actortype` structure suggests parallel or grouped effects. The linked-list structure (prev/next) enables insertion, removal, and traversal during cinematic playback.

## External Dependencies
- Standard C types: `int`, `char`, `byte`, `boolean` (defined elsewhere in project)
