# audiolib/source/awe32.h

## File Purpose
Public header file that defines the interface for the AWE32 Sound Blaster synthesizer driver. Provides error codes and MIDI control functions for initializing and operating the AWE32 audio chip, which was a common Sound Blaster soundcard used in 1990s PC games for MIDI music playback and sound synthesis.

## Core Responsibilities
- Define error codes for AWE32 operations
- Declare device initialization and shutdown
- Declare MIDI note on/off and note control functions
- Declare MIDI channel control (aftertouch, program change, control change, pitch bend)
- Provide error-to-string conversion

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| AWE32_ERRORS | enum | Error codes: Warning, Error, Ok, device detection/initialization errors, MPU-401 and DPMI errors |

## Global / File-Static State
None.

## Key Functions / Methods
All are public function declarations:

### AWE32_Init
- Signature: `int AWE32_Init(void)`
- Purpose: Initialize and detect the AWE32 device
- Outputs/Return: Status code from AWE32_ERRORS enum
- Notes: Must be called before any other AWE32 functions

### AWE32_Shutdown
- Signature: `void AWE32_Shutdown(void)`
- Purpose: Shut down the AWE32 device and release resources
- Notes: Should be called at game exit

### AWE32_NoteOn, AWE32_NoteOff
- Signature: `void AWE32_NoteOn(int channel, int key, int velocity)` / `void AWE32_NoteOff(int channel, int key, int velocity)`
- Purpose: MIDI note on/off events; controls note playback on a synth channel
- Inputs: MIDI channel (0–15), key/pitch (0–127), velocity (0–127)

### AWE32_ProgramChange
- Signature: `void AWE32_ProgramChange(int channel, int program)`
- Purpose: Select instrument/voice for a MIDI channel
- Inputs: MIDI channel, program number (0–127)

### AWE32_ControlChange
- Signature: `void AWE32_ControlChange(int channel, int number, int value)`
- Purpose: Send MIDI CC message (volume, pan, modulation, etc.)
- Inputs: MIDI channel, CC number, CC value (0–127)

### AWE32_PitchBend
- Signature: `void AWE32_PitchBend(int channel, int lsb, int msb)`
- Purpose: Adjust pitch on a channel
- Inputs: MIDI channel, LSB and MSB of 14-bit pitch value

### AWE32_PolyAftertouch, AWE32_ChannelAftertouch
- Signature: `void AWE32_PolyAftertouch(int channel, int key, int pressure)` / `void AWE32_ChannelAftertouch(int channel, int pressure)`
- Purpose: MIDI aftertouch (pressure) control per-note or per-channel
- Inputs: MIDI channel, optional key, pressure (0–127)

### AWE32_ErrorString
- Signature: `char *AWE32_ErrorString(int ErrorNumber)`
- Purpose: Convert error code to human-readable string
- Inputs: Error code from AWE32_ERRORS
- Outputs/Return: Pointer to error message string

## Control Flow Notes
This header is part of the audio subsystem init sequence. `AWE32_Init()` is called at engine startup; MIDI functions are called during gameplay to control music/synthesis; `AWE32_Shutdown()` is called at exit.

## External Dependencies
None (header-only declarations). Implementation in awe32.c would directly interface with AWE32 hardware or DOS/Windows DPMI and MPU-401 MIDI interfaces.
