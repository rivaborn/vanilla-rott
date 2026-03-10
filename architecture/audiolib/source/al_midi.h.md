# audiolib/source/al_midi.h

## File Purpose
Header file for the audio library's MIDI/FM synthesis (Adlib) interface. Defines error codes, hardware constants, and function declarations for voice management, MIDI note/control operations, and hardware initialization on legacy FM synthesizer cards.

## Core Responsibilities
- Define error codes and MIDI/audio constants (volume ranges, pitch bend, hardware port)
- Declare voice allocation and release functions
- Declare MIDI event handlers (note on/off, program change, control change)
- Declare hardware initialization, detection, and shutdown
- Declare stereo configuration and timbre bank loading
- Declare low-level register I/O to hardware

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `AL_Errors` | enum | Return codes for audio library operations |

## Global / File-Static State
None.

## Key Functions / Methods

### AL_Init
- Signature: `int AL_Init(int soundcard)`
- Purpose: Initialize audio hardware
- Inputs: soundcard type identifier
- Outputs/Return: Error code (`AL_Errors`)
- Side effects: Initializes FM synthesizer hardware
- Calls: Not inferable from this file

### AL_Shutdown
- Signature: `void AL_Shutdown(void)`
- Purpose: Shut down audio hardware and cleanup
- Inputs: None
- Outputs/Return: None
- Side effects: Releases hardware resources
- Calls: Not inferable from this file

### AL_NoteOn / AL_NoteOff
- Signature: `void AL_NoteOn(int channel, int key, int vel)` / `void AL_NoteOff(int channel, int key, int velocity)`
- Purpose: Start/stop a note on a MIDI channel
- Inputs: MIDI channel, key (pitch), velocity
- Outputs/Return: None
- Side effects: Triggers/stops voice on hardware
- Calls: Not inferable from this file

### AL_ReserveVoice / AL_ReleaseVoice
- Signature: `int AL_ReserveVoice(int voice)` / `int AL_ReleaseVoice(int voice)`
- Purpose: Allocate/deallocate a hardware voice
- Inputs: Voice number
- Outputs/Return: Error code or voice handle
- Side effects: Manages voice pool state
- Calls: Not inferable from this file

### AL_ProgramChange
- Signature: `void AL_ProgramChange(int channel, int patch)`
- Purpose: Select instrument for MIDI channel
- Inputs: MIDI channel, patch/program number
- Outputs/Return: None
- Side effects: Changes instrument on channel
- Calls: Not inferable from this file

### AL_ControlChange / AL_SetPitchBend
- Signature: `void AL_ControlChange(int channel, int type, int data)` / `void AL_SetPitchBend(int channel, int lsb, int msb)`
- Purpose: Apply MIDI continuous control or pitch modulation
- Inputs: Channel, control type/pitch data (14-bit)
- Outputs/Return: None
- Side effects: Modulates hardware synthesis parameters
- Calls: Not inferable from this file

### AL_DetectFM
- Signature: `int AL_DetectFM(void)`
- Purpose: Detect presence of FM synthesizer hardware
- Inputs: None
- Outputs/Return: Detection status (likely error code)
- Side effects: Hardware detection probing
- Calls: Not inferable from this file

### AL_RegisterTimbreBank
- Signature: `void AL_RegisterTimbreBank(unsigned char *timbres)`
- Purpose: Load custom instrument definitions into synthesizer
- Inputs: Pointer to timbre data buffer
- Outputs/Return: None
- Side effects: Initializes synthesizer instrument parameters
- Calls: Not inferable from this file

**Notes on helpers:**
- `AL_SendOutputToPort`, `AL_SendOutput` – low-level hardware register I/O
- `AL_StereoOn`, `AL_StereoOff` – stereo output control
- `AL_AllNotesOff` – panic/stop all notes on channel
- `AL_SetMaxMidiChannel` – limit active MIDI channels
- `AL_Reset` – software reset

## Control Flow Notes
Typical initialization → detection → timbre load → voice management → MIDI playback → shutdown sequence. This is a hardware interface layer; actual MIDI sequencing logic would be in a higher-level module.

## External Dependencies
- No includes (header-only declarations)
- Assumes caller provides MIDI channel/voice numbers and hardware port mappings
- Hardware: Adlib/FM synthesizer at port `0x388`
