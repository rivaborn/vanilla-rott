# rott/_rt_buil.h

## File Purpose
Private header file for Rise of the Triad's build/rendering subsystem. Defines compile-time constants and data structures for managing textured planes, menu layout, and viewport rendering parameters.

## Core Responsibilities
- Define texture rendering constants (dimensions, scaling parameters)
- Specify menu UI layout constants (offsets, title positioning)
- Declare the `plane_t` structure for representing renderable geometric planes
- Provide utility macros (MAX, MAXPLANES)
- Configure fixed-resolution rendering pipeline parameters

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| plane_t | struct | Represents a screen-space rectangle with texture mapping (coordinates, texture index, scaling) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| TEXTUREW | macro | compile-time | Base texture width constant (288) |
| TEXTUREWIDTH | macro | compile-time | Scaled texture width: (TEXTUREW × 1024) − 1 |
| TEXTUREHEIGHT | macro | compile-time | Texture height constant (158) |
| NORMALVIEW | macro | compile-time | Standard viewport configuration flag (0x40400L) |
| NORMALHEIGHTDIVISOR | macro | compile-time | Fixed-point height scaling divisor (156000000) |
| NORMALWIDTHMULTIPLIER | macro | compile-time | Fixed-point width scaling multiplier (241) |
| MENUOFFY | macro | compile-time | Menu Y-offset (10 pixels) |
| MENUTITLEY | macro | compile-time | Menu title Y-position (10 pixels) |
| MENUBACKNAME | macro | compile-time | Background plane identifier string ("plane") |
| MAXPLANES | macro | compile-time | Maximum planes per frame (10) |
| FLIPTIME | macro | compile-time | Animation frame interval (20 ticks) |

## Key Functions / Methods
None. Header-only definitions file.

## Control Flow Notes
This file configures the fixed-resolution rendering pipeline (288×158 texture resolution). Constants are evaluated at compile-time; `plane_t` instances are likely allocated and manipulated during map initialization and per-frame rendering. The scaling multiplier/divisor pair suggests integer-based or lookup-table-driven coordinate transformations rather than floating-point math.

## External Dependencies
- Included by runtime build/rendering modules (not inferable from this file)
- No external includes or dependencies visible

## Notes
- Comment "Should be 10 with titles" suggests MENUOFFY was designed with title bars in mind
- Large NORMALHEIGHTDIVISOR (156M) relative to small multiplier (241) indicates aggressive integer quantization for fixed-point geometry
- FLIPTIME (20) suggests ~33ms frame timing at 60 FPS (60÷20 = 3 frames per flip)
