# audiolib/source/oldtimbr.c

## File Purpose
Provides OPL chip FM synthesis timbre (instrument) presets for a retro game audio library. Contains configuration data for 174 instrument timbres and a percussion mapping table for MIDI-to-sound routing.

## Core Responsibilities
- Define TIMBRE struct for FM operator parameters (SAVEK, envelope, waveform, feedback)
- Define DRUM_MAP struct for percussion note-to-timbre mapping
- Supply PercussionTable array mapping all 128 MIDI percussion notes to drum timbres
- Supply ADLIB_TimbreBank with 174 preset instrument configurations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TIMBRE | struct | FM synthesis parameters: two operator settings, two envelope stages, two waveforms, feedback level |
| DRUM_MAP | struct | Percussion entry: timbre bank index + key/pitch value |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| PercussionTable | DRUM_MAP[128] | global | MIDI percussion mapping; indices 0–35 mostly point to timbre 128; indices 36–88 use diverse timbres (129–173) |
| ADLIB_TimbreBank | TIMBRE[174] | global | FM synthesis presets; most entries use paired operator parameters; waveform and envelope values in 0–255 range |

## Key Functions / Methods
None.

## Control Flow Notes
Pure data file. At initialization, a loader likely copies these arrays into synthesizer state. At runtime, MIDI percussion events index PercussionTable to find a timbre and key; melodic events index ADLIB_TimbreBank directly. No runtime logic in this file.

## External Dependencies
- No includes
- Assumed consumed by audio driver code elsewhere in `audiolib/` that configures Yamaha OPL FM synthesizers
