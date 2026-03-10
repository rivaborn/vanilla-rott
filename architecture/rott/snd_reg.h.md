# rott/snd_reg.h

## File Purpose
Sound registry and configuration file that maps digital sound identifiers to engine sound parameters, priorities, and mixing flags. This is the primary sound lookup table for the game engine, organizing hundreds of game sound effects across categories like menu, weapons, player actions, enemies, and environmental effects.

## Core Responsibilities
- Define enum of all digital sound IDs used throughout the game
- Map each digital sound to its MUSE (sound subsystem) equivalent
- Assign sound priority levels for mixer arbitration (menu > game > secondary > environmental)
- Configure sound playback flags (pitch shift enabled/disabled, playonce, write, etc.)
- Organize sounds by functional category (menus, weapons, actors, environment, secrets)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `digisounds` | enum | Sound ID registry; values range from `D_MENUFLIP` to `D_LASTSOUND` |
| `sounds` | array | Static mapping of digital sounds to configuration tuples (MUSE ID, flags, priority) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sounds` | `sound_t[]` | file-static | Master sound configuration table indexed by digital sound ID; initialized at load time |

## Key Types / Data Structures
None to document as functions—this is pure data/configuration.

## Control Flow Notes
**Initialization phase:** The `sounds` array is consulted during engine initialization or first use to populate the sound mixer/driver with metadata. Sound queries during gameplay reference this table for priority, flags, and MUSE equivalents. Not involved in frame/update/render loops.

## External Dependencies
- **`sound_t`** — struct type containing digital ID, MUSE ID, flags, and priority (defined elsewhere)
- **`MAXSOUNDS`** — array size constant (defined elsewhere)
- **MUSE sound IDs** — e.g., `MUSE_MENUFLIPSND`, `MUSE_LASTSOUND` (defined elsewhere, likely in sound driver header)
- **Sound flag constants** — e.g., `SD_PITCHSHIFTOFF`, `SD_PLAYONCE`, `SD_WRITE` (defined elsewhere)
- **Priority constants** — e.g., `SD_PRIOMENU`, `SD_PRIOPCAUSD`, `SD_PRIOBOSS`, `SD_PRIOREMOTE` (defined elsewhere)

## Notes
- The enum contains ~300+ sound identifiers organized by gameplay context (weapons, actors, environment, etc.)
- Many sounds are commented out, suggesting legacy/unused entries.
- Flags and priorities suggest a priority-based mixer: menu sounds block game sounds; boss sounds block ambient; explosions block secondary effects.
- Several entries reuse MUSE IDs (e.g., multiple pistol variants map to `MUSE_ATKPISTOLSND`), indicating the audio engine normalizes similar sounds.
- Some mappings appear inconsistent or placeholder (e.g., remote sounds mapping to `MUSE_LASTSOUND`), suggesting incomplete implementation or cleanup.
