# audiolib/source/gmtemp.c

## File Purpose
Defines a bank of pre-configured AdLib OPL synthesizer patches (timbres) for FM synthesis. Contains 256 hardcoded TIMBRE entries that represent different instrument sounds and effects for the AdLib sound card.

## Core Responsibilities
- Provide a global timbre bank with 256 pre-initialized synthesizer patches
- Define the TIMBRE data structure for AdLib FM synthesis parameters
- Enable sound/music modules to access standardized instrument definitions
- Supply patch parameters for two operator FM synthesis (modulator and carrier)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TIMBRE | struct | AdLib FM synthesis patch definition with two-operator parameters |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| ADLIB_TimbreBank | TIMBRE[256] | global | Pre-initialized bank of 256 synthesizer patches for instrument/effect sounds |

## Key Functions / Methods
None. This is a data-only file.

## Control Flow Notes
This file is passive data initialization. The TIMBRE bank is likely loaded at engine startup and indexed by sound/music modules when selecting instrument patches during playback or initialization. No executable code is present.

## External Dependencies
- Standard C library includes (implicit via GPL license header)
- Likely consumed by AdLib driver or music/sound system files that reference `ADLIB_TimbreBank`

---

**Notes:**
- TIMBRE structure maps directly to AdLib/OPL hardware registers: `SAVEK`, `Level`, `Env1`, `Env2`, `Wave` correspond to operator parameters; `Feedback` controls algorithm/feedback; `Transpose` adjusts pitch
- Array entries 128–255 contain many repeated or blank entries (particularly indices 128–160 repeat the same patch, and indices 161+ have varied transpose values), suggesting incomplete or placeholder patch definitions
- This is typical of legacy DOS game audio—pre-computed patch tables avoid runtime synthesis overhead
