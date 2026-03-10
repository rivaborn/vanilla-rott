# audiolib/source/leeold.c

## File Purpose
Pure data file defining AdLib synthesizer instrument and percussion configurations. Provides 256 instrument timbres and 128 percussion note mappings for FM synthesis-based music and sound effects in the game engine.

## Core Responsibilities
- Store FM synthesis parameters (SAVEK, envelope, waveform, feedback) for 256 instruments
- Define percussion MIDI note-to-timbre mapping for drum sounds
- Serve as lookup tables for sound engine initialization and playback
- Encode instrument presets compatible with AdLib sound card hardware

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TIMBRE | struct | FM synthesis operator parameters: envelope rates, waveforms, feedback control |
| DRUM_MAP | struct | Maps MIDI percussion notes (0–127) to timbre index and pitch offset |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| PercussionTable | `DRUM_MAP[128]` | global | MIDI percussion note-to-timbre mapping; most entries map to timbre 128 (disabled) |
| ADLIB_TimbreBank | `TIMBRE[256]` | global | 256 instrument timbres with FM synthesis parameters; majority populate range 0–100+ with varied synth configs |

## Key Functions / Methods
None.

## Control Flow Notes
Pure static data. `ADLIB_TimbreBank` and `PercussionTable` are loaded during engine initialization for use by the sound/music subsystem during playback. `PercussionTable` is indexed by MIDI note number; `ADLIB_TimbreBank` is indexed by instrument/timbre ID.

## External Dependencies
- Standard C types only (`unsigned char`).
- No includes or external function calls.
