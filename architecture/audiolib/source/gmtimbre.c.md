# audiolib/source/gmtimbre.c

## File Purpose
Provides a static timbre (instrument) bank for the AdLib sound card, containing 256 FM synthesizer parameter configurations for audio synthesis. This is a direct data table mapping MIDI instruments to hardware synthesizer register settings.

## Core Responsibilities
- Defines the TIMBRE struct encoding FM operator parameters (envelope, level, waveform)
- Supplies a hardcoded lookup table of 256 instrument definitions for AdLib compatibility
- Encodes MIDI-related metadata (transpose, velocity sensitivity) per instrument

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TIMBRE | struct | FM synthesizer parameters for one instrument: two operators with key scaling, levels, envelopes, waveforms, plus feedback/algorithm and MIDI transpose/velocity |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| ADLIB_TimbreBank | TIMBRE[256] | global | Lookup table of 256 predefined instrument configurations indexed by MIDI program number |

## Key Functions / Methods
None. This is a data-only initialization file.

## Control Flow Notes
Loaded at startup as a static timbre bank. Referenced when synthesizing sound to configure FM synthesizer hardware registers for the selected instrument. Not part of frame/render loops.

## External Dependencies
None. This file contains only struct definition and data initialization.

## Notes
- TIMBRE struct packs 8-bit fields directly matching AdLib FM synthesizer register layout (operators 0 and 1 parameters)
- Entries 0–127: General MIDI instruments with mostly zero Velocity values
- Entries 128–191: Primarily zero or repeated patterns with non-zero Velocity (35, 48, 58, 60, etc.), suggesting percussion/drum kits
- Entries 192–255: Many duplicate "default" entries (16, 17, 68, 0, 248, 243, 119, 6, 2, 0, 8, 35)
