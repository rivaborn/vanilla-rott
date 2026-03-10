# rott/snd_shar.h

## File Purpose
Sound enumeration and static sound table for the ROTT game engine. Maps ~300+ game sound events to digital audio samples and MUSE music system equivalents, with playback priority and behavior flags.

## Core Responsibilities
- Defines `digisounds` enum with all sound event IDs used throughout the game
- Initializes static `sounds[]` array mapping each sound to digital/MUSE variants and priority
- Organizes sounds by category (menu, weapons, player, enemies, environment, secrets)
- Specifies sound playback priorities to manage audio resource contention
- Documents pitch-shift and play-once behavior for specific sounds

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `digisounds` | enum | Sound event identifiers (D_MENUFLIP, D_ATKPISTOLSND, etc.) |
| `sound_t` | typedef (inferred) | Sound record: {digital_id, muse_id}, flags, priority (defined elsewhere) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sounds` | `sound_t[MAXSOUNDS]` | static | Master sound lookup table indexed by `digisounds` enum; maps each game sound to audio assets and playback rules |

## Key Functions / Methods
None. This file is declarative data only.

## Control Flow Notes
Initialization phase: The `sounds[]` table is compiled as a static constant, likely loaded at engine startup. At runtime, other modules index this table by `digisounds` enum values to retrieve playback properties (digital sample, MUSE equivalent, priority level, flags). Priority system manages simultaneous sound playback; higher-priority categories (e.g., `SD_PRIOBOSS`) override lower ones (e.g., `SD_PRIOSECRET`).

## External Dependencies
- **Undefined here (defined elsewhere):**
  - `sound_t` typedef – likely `snd_shar.c` or header
  - `D_*` constants – digital sound sample IDs
  - `MUSE_*` constants – MUSE music system sound IDs
  - `SD_*` constants – flags (`SD_WRITE`, `SD_PLAYONCE`, `SD_PITCHSHIFTOFF`) and priority levels (`SD_PRIOMENU`, `SD_PRIOGAME`, `SD_PRIOBOSS`, etc.)
  - `MAXSOUNDS` – array size constant

## Notes
- Many entries map to `D_LASTSOUND` / `MUSE_LASTSOUND`, indicating placeholder/unimplemented sounds.
- Dual-source design: each sound has a digital sample fallback and a MUSE synthesizer equivalent.
- Priority tiers support complex hierarchies: player actions > boss sounds > actor sounds > environment > secrets.
- Sound categories inferred from grouped enum entries: menu, gameplay, weapons, actors (guards, monks, robots, bosses), pickups, hazards.
