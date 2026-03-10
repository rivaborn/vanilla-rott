# rott/sprites.h

## File Purpose
Defines enumerated sprite identifiers for all actors, enemies, effects, and weapons in the ROTT engine. This is a centralized sprite ID registry used by game logic and rendering systems to reference animations and visual assets.

## Core Responsibilities
- Define sprite IDs for enemy types (guards, monks, bosses) with their animation sequences (standing, walking, shooting, pain, death)
- Define sprite IDs for hazards and environmental effects (blades, fire jets, crushers, explosions, gibs)
- Define sprite IDs for power-ups, collectibles, and interactive objects
- Define weapon sprite IDs and animations
- Provide sprite aliases for code reuse (e.g., fallback sprites, shared animations)
- Organize sprite IDs by category with clear naming conventions (prefixes like `SPR_`, `W_`, actor names)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `actornames_t` | enum | Master enumeration of all sprite IDs for actors, effects, hazards, and collectibles; ordered to match sprite resource order |
| `weaponsprites` | enum | Enumeration of weapon-related sprite frames for different character types and weapon states |

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This file is purely declarative—it provides no control flow. It is included by actor AI/rendering systems and the sprite manager to translate logical sprite names into array indices or resource identifiers. The enum order matters for sprite resource mapping.

## External Dependencies
- **develop.h**: Provides compilation flags (`SHAREWARE`, `SHAREWARE == 0`) to conditionally include/exclude shareware-restricted sprites

## Notes
- Sprites are organized by character/enemy type with comment headers (e.g., `UNTERWACHE`, `ANGRIFFSTUPPE`, `BLITZWACHE`)
- Naming patterns: `SPR_<ACTOR>_<ACTION><DIRECTION><FRAME>` (e.g., `SPR_LOWGRD_W11` = Lowguard walking direction 1, frame 1)
- Walking sequences follow a consistent 4-direction, 8-frame pattern (W1–W4 for directions, sub-frames 1–8)
- Many commented-out sprite definitions indicate removed/unused features
- Sprite aliases at the end use `#define` for fallbacks (e.g., pain sprites mapped to death frames) and hit detection dummies (`-1`)
- Conditional sprites (prefixed with `W_`) depend on `SHAREWARE == 0` for full-version content
