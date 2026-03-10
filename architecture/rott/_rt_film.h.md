# rott/_rt_film.h

## File Purpose
Defines structures and constants for a film/demo/cutscene playback system. Supports timestamped events with visual properties and actor state tracking across sequential frames.

## Core Responsibilities
- Define event type enumerations (backgrounds, sprites, palettes, fades)
- Specify event properties (position, scale, velocity, animation length)
- Track actor state during film playback (current position/scale, event index)
- Set capacity limits for events and actors per film sequence

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `eventtype` | enum | Classifies event categories: backgrounds, sprite layers, palette changes, fades |
| `event` | struct | Single timestamped event; contains visual properties (position, scale, velocity), asset reference, and animation metadata |
| `actortype` | struct | Runtime state for an actor during playback; tracks elapsed tics, current event index, and interpolated position/scale |

## Global / File-Static State
None.

## Key Functions / Methods
None—header defines only types and constants.

## Control Flow Notes
Inferred as part of a demo/film playback pipeline:
1. Events are sequenced chronologically by `time` field
2. Actors iterate through events as playback progresses (`eventnumber` index)
3. Current state (`curx`, `cury`, `curscale`) updated via interpolation between event properties and velocity deltas
4. Directional flag (`dir`: LEFT/RIGHT) likely controls sprite orientation

## External Dependencies
Standard C types only; no external includes visible.
