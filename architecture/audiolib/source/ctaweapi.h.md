# audiolib/source/ctaweapi.h

## File Purpose
Header file defining the Sound Blaster AWE32 audio interface API for DOS. Declares hardware control functions, MIDI/NRPN support, SoundFont management, and device state variables. Compiled for multiple C compilers with platform-specific memory and calling conventions.

## Core Responsibilities
- Define cross-compiler type aliases and macros (BYTE, WORD, DWORD, FAR pointers)
- Declare hardware register access functions (awe32RegW, awe32RegRW, etc.)
- Export MIDI support functions (note on/off, program change, controllers, pitch bend)
- Export NRPN (Non-Registered Parameter Number) initialization
- Declare SoundFont and wave packet streaming functions for sample data loading
- Define data structures (SOUND_PACKET, WAVE_PACKET) for metadata during streaming
- Manage struct packing and name mangling directives for __WATCOMC__, __HIGHC__, __SC__ compilers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SOUND_PACKET | struct | Metadata for SoundFont bank loading (sizes, file offset, buffer pointers, patch RAM info) |
| WAVE_PACKET | struct | Metadata for wave sample streaming (sample rate, bit depth, loop points, channels) |
| SOUNDPAD | struct | Container for 7 FAR byte pointers to sound pad patch data |
| SCRATCH | typedef char[702] | Scratch buffer for temporary data |
| SOUNDFONT | typedef char[124] | SoundFont object buffer |
| GCHANNEL | typedef char[20] | Global MIDI channel state |
| MIDICHANNEL | typedef char[32] | Per-channel MIDI state |
| NRPNCHANNEL | typedef char[96] | Per-channel NRPN state |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| awe32NumG | WORD | global | Number of AWE32 voices available |
| awe32BaseAddx | WORD | global | Base hardware I/O port address |
| awe32DramSize | DWORD | global | Total DRAM on sound card (bytes) |
| awe32Scratch | SCRATCH | global | Temporary working buffer |
| awe32SFont[4] | SOUNDFONT array | global | 4 active SoundFont banks |
| awe32GChannel[32] | GCHANNEL array | global | 32 global channel states |
| awe32MIDIChannel[16] | MIDICHANNEL array | global | 16 MIDI input channel states |
| awe32NRPNChannel[16] | NRPNCHANNEL array | global | 16 NRPN parameter states per channel |
| awe32SoundPad | SOUNDPAD | global | Sound pad patch data pointers |
| awe32SPad1Obj–awe32SPad7Obj | BYTE arrays | global | 7 sound pad patch objects |

## Key Functions / Methods

### awe32RegW, awe32RegRW, awe32RegDW, awe32RegRDW
- Signature: `void PASCAL awe32RegW(WORD port, WORD value)`, `WORD PASCAL awe32RegRW(WORD port)`, `void PASCAL awe32RegDW(WORD port, DWORD value)`, `DWORD PASCAL awe32RegRDW(WORD port)`
- Purpose: Low-level hardware register write/read (8-bit, 16-bit, 32-bit variants)
- Inputs: Hardware port number, optional value to write
- Outputs/Return: Current register value (read variants)
- Side effects: Direct hardware I/O
- Calls: None visible
- Notes: Foundation for all hardware communication

### awe32InitMIDI
- Signature: `WORD PASCAL awe32InitMIDI(void)`
- Purpose: Initialize MIDI subsystem and voice allocation
- Outputs/Return: Status code (WORD)
- Side effects: Populates awe32MIDIChannel global state
- Calls: None visible
- Notes: Must be called after awe32InitHardware

### awe32NoteOn, awe32NoteOff, awe32ProgramChange, awe32Controller, etc.
- Signature: `WORD PASCAL awe32NoteOn(WORD channel, WORD note, WORD velocity)`, similar for others
- Purpose: MIDI event handlers (note triggering, releases, program selection, continuous controllers, pressure, pitch bend)
- Inputs: MIDI channel (0–15), note/CC number, velocity/value
- Outputs/Return: Status code
- Side effects: Hardware voice allocation, register writes
- Calls: Likely awe32RegW/awe32RegDW internally
- Notes: __awe32NoteOff and __awe32IsPlaying are internal variants with extra parameters

### awe32Detect
- Signature: `WORD PASCAL awe32Detect(WORD port)`
- Purpose: Probe hardware at given I/O port; confirm AWE32 presence
- Inputs: Base port address to test
- Outputs/Return: Status code (0=success)
- Side effects: None (read-only)
- Calls: Likely register reads

### awe32InitHardware, awe32Terminate
- Signature: `WORD PASCAL awe32InitHardware(void)`, `WORD PASCAL awe32Terminate(void)`
- Purpose: Initialize/shutdown hardware; reset voice state and memory
- Outputs/Return: Status code
- Side effects: Hardware initialization/reset; populates awe32NumG, awe32DramSize, awe32BaseAddx
- Notes: awe32Detect should precede awe32InitHardware

### awe32SFontLoadRequest, awe32StreamSample, awe32SetPresets, awe32ReleaseBank
- Signature: `WORD PASCAL awe32SFontLoadRequest(SOUND_PACKET FAR*)`, etc.
- Purpose: Multi-step SoundFont loading pipeline (request allocation, stream sample data, configure presets, release)
- Inputs: Pointer to SOUND_PACKET or WAVE_PACKET with metadata and data buffer
- Outputs/Return: Status code
- Side effects: Hardware memory writes, voice allocation
- Notes: Forms a state machine; sample streaming done in packets

### awe32InitNRPN
- Signature: `WORD PASCAL awe32InitNRPN(void)`
- Purpose: Initialize NRPN (extended parameter) subsystem
- Outputs/Return: Status code
- Side effects: Populates awe32NRPNChannel global state

## Control Flow Notes
This header declares a library API; implementations are external. Typical initialization order: `awe32Detect()` → `awe32InitHardware()` → `awe32InitMIDI()` / `awe32InitNRPN()` → then MIDI events (`awe32NoteOn()` etc.) and sound loading (`awe32SFontLoadRequest()` → `awe32StreamSample()` → `awe32SetPresets()`). Shutdown: `awe32Terminate()`.

## External Dependencies
- **Compiler directives**: `__FLAT__`, `__HIGHC__`, `DOS386`, `__SC__`, `__WATCOMC__` for struct packing and calling conventions
- **All functions and variables declared `extern`**: Implementations in linked modules (`__midieng_code`, `__hardware_code`, `__sbkload_code`, `__nrpn_code`)
- **No standard library includes** (pure hardware API)
