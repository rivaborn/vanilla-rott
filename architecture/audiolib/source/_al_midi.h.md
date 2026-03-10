# audiolib/source/_al_midi.h

## File Purpose
Header file defining MIDI and AdLib FM synthesizer constants, data structures, and function declarations for the audio library. Establishes the interface for managing voices, channels, and instrument timbres on the AdLib sound card, a common DOS-era music synthesizer.

## Core Responsibilities
- Define MIDI message types, note values, and control constants
- Define hardware register constants for AdLib chip control
- Declare data structures for voice and channel state management
- Declare timbre (instrument) storage and retrieval
- Provide utility macros for byte manipulation and frequency calculation
- Declare voice and channel control functions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `VOICE` | struct | Represents a single playing voice: note, velocity, channel, pitch bend, timbre, and port information |
| `VOICELIST` | struct | Doubly-linked list container (start/end pointers) for managing voices |
| `CHANNEL` | struct | Represents a MIDI channel with volume, pan, pitch bend, detune, and active voices |
| `TIMBRE` | struct | Instrument parameters: envelopes, waveform, feedback, velocity curve, transposition |
| `cromatic_scale` | enum | Note frequency values in hex (C through B_SHARP) for pitch generation |
| `octaves` | enum | Octave offset masks (0–7) ORed with F-Number for frequency calculation |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ADLIB_TimbreBank` | `TIMBRE[256]` | extern | Pre-loaded instrument definitions indexed by program number |

## Key Functions / Methods

### AL_ResetVoices
- Signature: `static void AL_ResetVoices(void)`
- Purpose: Initialize or clear all voice state
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies global voice state
- Calls: (not inferable)
- Notes: Called during initialization or when flushing all audio

### AL_AllocVoice
- Signature: `static int AL_AllocVoice(void)`
- Purpose: Allocate a free voice slot for playing a new note
- Inputs: None
- Outputs/Return: Voice index or `AL_VoiceNotFound` (-1) if none available
- Side effects: Updates voice allocation state
- Calls: (not inferable)
- Notes: 18 chip slots; may reuse oldest voice if all full

### AL_GetVoice
- Signature: `static int AL_GetVoice(int channel, int key)`
- Purpose: Find or create a voice for a specific channel and note key
- Inputs: MIDI channel (0–15), note key (0–127)
- Outputs/Return: Voice index
- Side effects: May allocate new voice
- Calls: (not inferable)
- Notes: Used during note-on events

### AL_SetVoiceTimbre
- Signature: `static void AL_SetVoiceTimbre(int voice)`
- Purpose: Load instrument parameters for a voice from the timbre bank
- Inputs: Voice index
- Outputs/Return: None
- Side effects: Writes to AdLib hardware registers
- Calls: (not inferable)
- Notes: Pulled from `ADLIB_TimbreBank` based on channel program

### AL_SetVoicePitch, AL_SetVoiceVolume
- Purpose: Configure pitch (via F-Number + octave) and volume envelope for a voice
- Inputs: Voice index (and optional parameters like detune for pitch)
- Outputs/Return: None
- Side effects: Write AdLib chip registers
- Notes: Called continuously during note sustain; pitch affected by channel detune and pitch bend

### AL_SetChannelVolume, AL_SetChannelPan, AL_SetChannelDetune
- Purpose: Apply channel-wide settings (volume, stereo pan, detuning) to all active voices on that channel
- Inputs: Channel index, and new value (volume 0–127, pan, detune)
- Outputs/Return: None
- Side effects: Update `CHANNEL` struct; may affect all voices
- Calls: (not inferable)

### AL_CalcPitchInfo
- Signature: `static void AL_CalcPitchInfo(void)`
- Purpose: Pre-compute pitch bend and octave offsets based on channel settings
- Inputs: None
- Outputs/Return: None
- Side effects: Updates pitch lookup tables
- Calls: (not inferable)
- Notes: Likely called when pitch bend or detune changes

## Control Flow Notes
This file is a header; actual control flow resides in `_al_midi.c`. The interface suggests a frame/update loop:
1. **Init**: `AL_ResetVoices()`, load `ADLIB_TimbreBank`
2. **Note On**: `AL_GetVoice()`, `AL_SetVoiceTimbre()`, `AL_SetVoicePitch()`
3. **Note Off**: Voice deallocation (not declared here)
4. **Update**: `AL_SetVoicePitch()` per voice (for pitch bend), `AL_SetChannelVolume/Pan/Detune()` per channel
5. **Render**: Hardware register writes (inside the above functions)

## External Dependencies
- **Defined elsewhere**: `ADLIB_TimbreBank` (defined in `_al_midi.c`)
- **Hardware constants**: `alFreqH` (0xb0), `alEffects` (0xbd) — AdLib register addresses
- **Macros**: `lobyte`, `hibyte` for 16-bit register pair splitting
- No C library dependencies visible; bare hardware abstraction
