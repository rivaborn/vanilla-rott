# audiolib/source/leetimb1.c

## File Purpose
Data definition file containing timbre (FM synthesis instrument) configurations and percussion mappings for the AdLib FM audio engine. Provides lookup tables for 256 instrument timbres and 128 MIDI percussion drum sounds used during audio synthesis and music playback.

## Core Responsibilities
- Define FM synthesis timbre parameters (envelope, waveform, feedback settings)
- Provide percussion-to-timbre mapping for MIDI drum notes (channels 120–127)
- Store complete AdLib timbre bank with factory presets for game instruments
- Support initialization of audio synthesis engine with instrument definitions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TIMBRE | struct | FM synthesis parameters: two-operator oscillator config (SAVEK, Level, Env1, Env2, Wave per operator), plus feedback/modulation |
| DRUM_MAP | struct | MIDI percussion mapping: Timbre index + Key offset for drum sounds |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| PercussionTable | DRUM_MAP[128] | global | Maps MIDI notes 0–127 to percussion timbres and key values; most entries unmapped (128, 0) |
| ADLIB_TimbreBank | TIMBRE[256] | global | Complete FM timbre library with 256 instrument configurations; many indices repeat default settings |

## Key Functions / Methods
None.

## Control Flow Notes
This file is a pure data resource loaded once at audio engine initialization. The tables are referenced during:
- **Music playback**: ADLIB_TimbreBank provides instrument definitions for melody/harmony voices
- **Percussion synthesis**: PercussionTable maps incoming MIDI drum events to the correct timbre and pitch offset
- **FM register configuration**: TIMBRE fields directly configure AdLib hardware registers (20h–35h range for operators)

Not part of a traditional init/frame/render cycle—consumed by audio synthesis functions as needed.

## External Dependencies
- **Standard C types only** (unsigned char, implicit array sizing)
- **Defined elsewhere**: Functions that read these tables (audio driver, MIDI playback engine) not visible in this file
- **No external includes**
