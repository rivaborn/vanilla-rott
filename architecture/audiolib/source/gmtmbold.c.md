# audiolib/source/gmtmbold.c

## File Purpose
Defines AdLib FM synthesis timbre data and MIDI percussion-to-timbre mappings for a game audio system. Contains pre-configured instrument parameters and drum note lookup tables used during MIDI playback.

## Core Responsibilities
- Provide 174 pre-configured FM synthesis timbres/instruments for AdLib sound generation
- Map MIDI percussion note numbers (0–127) to appropriate drum timbres and key values
- Supply FM operator parameters (envelope, waveform, feedback, sustain levels) for both operators
- Support dynamic instrument selection during MIDI playback

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TIMBRE | struct | FM synthesis operator configuration; two pairs of SAVEK (sustain/attack), Level, Env (envelope), Wave (waveform), and Feedback for two-operator FM voices |
| DRUM_MAP | struct | MIDI percussion note mapping; pairs a timbre bank index with a MIDI key value |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| PercussionTable | DRUM_MAP[128] | global | Lookup table mapping MIDI percussion notes (0–127) to timbre indices and key values; many default entries at boundaries |
| ADLIB_TimbreBank | TIMBRE[174] | global | Pre-computed FM synthesis parameters for 174 instrument timbres; used by audio engine during note playback |

## Key Functions / Methods
None — file contains only data definitions.

## Control Flow Notes
This file is purely declarative data. At audio engine startup, these tables are likely loaded into memory. During MIDI playback:
1. A percussion note (0–127) indexes `PercussionTable` to find a timbre ID and key offset
2. The timbre ID indexes `ADLIB_TimbreBank` to fetch FM synthesis operator parameters
3. These parameters configure the AdLib/OPL2 synthesizer hardware to render the sound

## External Dependencies
- No includes or external symbols — self-contained data module
- Implicitly used by other audio subsystem code that performs MIDI→AdLib synthesis conversion

## Notes
- Many `PercussionTable` entries are duplicates (128, 35), likely representing unmapped or default percussion slots
- FM parameters appear optimized for Yamaha OPL2/OPL3 chipset (two-operator FM synthesis)
- File naming ("gmtmbold") suggests "General MIDI Timbre Bold" instrument set
