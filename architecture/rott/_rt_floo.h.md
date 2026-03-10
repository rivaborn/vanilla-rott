# rott/_rt_floo.h

## File Purpose
Private header file defining compile-time constants for floor and sky rendering in the ray-traced renderer. Establishes limits on view dimensions, sky segment geometry, and rendering thresholds.

## Core Responsibilities
- Define maximum view height constant (linked to global screen height)
- Set hard limit on sky segment count (geometry optimization)
- Establish sky data structure size constraint
- Define minimum sky height threshold for rendering decisions

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MAXVIEWHEIGHT | macro (int) | compile-time | Maximum rendering view height; aliases MAXSCREENHEIGHT |
| MAXSKYSEGS | macro (int) | compile-time | Hard limit of 2048 sky segments for geometry batching |
| MAXSKYDATA | macro (int) | compile-time | Fixed sky data array size of 8 elements |
| MINSKYHEIGHT | macro (int) | compile-time | Minimum sky height (148 pixels) for culling/rendering threshold |

## Key Functions / Methods
None.

## Control Flow Notes
This file operates at compile-time only. Constants are used by the rendering pipeline (likely in raycasting/wall/sky rendering loops) to allocate buffers and enforce geometric limits. The minimum sky height is likely a culling threshold in the main render loop.

## External Dependencies
- `MAXSCREENHEIGHT` (defined elsewhere) — referenced by MAXVIEWHEIGHT macro, indicating coupling to global screen/viewport configuration
- Standard C preprocessor (no function library includes)
