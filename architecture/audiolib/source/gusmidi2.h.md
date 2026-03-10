# audiolib/source/gusmidi2.h

## File Purpose
Public interface header for Gravis Ultrasound (GUS) MIDI synthesizer support. Declares functions for initializing/shutting down the GUS system, loading instrument patches, and sending MIDI events (note on/off, control change, pitch bend, program change).

## Core Responsibilities
- Define GUS error codes and translate to human-readable strings
- Initialize and shut down GUS hardware and MIDI subsystems
- Load and unload instrument patches into GUS DRAM
- Send MIDI control messages (note on/off, pitch bend, program change, control change)
- Manage master volume for the GUS output
- Allocate DOS-accessible memory for GUS operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GUS_Errors` | enum | Error codes returned by GUS/GUSMIDI functions; range from `GUS_Warning` (−2) to `GUS_MissingConfig` (11) |

## Global / File-Static State
None.

## Key Functions / Methods

### GUS_Init / GUS_Shutdown
- Signature: `int GUS_Init(void)` / `void GUS_Shutdown(void)`
- Purpose: Initialize and tear down the GUS hardware interface
- Inputs: None
- Outputs/Return: `GUS_Init` returns error code (`GUS_Ok` on success)
- Side effects: Hardware initialization/reset; may allocate memory
- Calls: Not inferable from this file
- Notes: Must be called before GUSMIDI_Init; inverse pairing expected

### GUSMIDI_Init / GUSMIDI_Shutdown
- Signature: `int GUSMIDI_Init(void)` / `void GUSMIDI_Shutdown(void)`
- Purpose: Initialize and tear down the MIDI event dispatch layer on top of GUS
- Inputs: None
- Outputs/Return: `GUSMIDI_Init` returns error code
- Side effects: Initializes MIDI channel/voice state; depends on GUS being initialized first
- Calls: Not inferable from this file
- Notes: Typically called after `GUS_Init`

### GUSMIDI_NoteOn / GUSMIDI_NoteOff
- Signature: `void GUSMIDI_NoteOn(int chan, int note, int velocity)` / `void GUSMIDI_NoteOff(int chan, int note, int velocity)`
- Purpose: Trigger or release a note on a MIDI channel
- Inputs: Channel (0–15), MIDI note number (0–127), velocity (0–127)
- Outputs/Return: None
- Side effects: Allocates voice on GUS; triggers sound synthesis
- Calls: Not inferable from this file
- Notes: `NoteOff` with velocity=0 is equivalent to `NoteOn` with velocity=0

### GUSMIDI_ProgramChange
- Signature: `void GUSMIDI_ProgramChange(int channel, int prog)`
- Purpose: Select instrument patch for a MIDI channel
- Inputs: Channel (0–15), program number (0–127)
- Outputs/Return: None
- Side effects: Loads patch into voice memory if not already loaded
- Calls: Likely calls `GUSMIDI_LoadPatch`
- Notes: Patch must exist via `GUS_GetPatchMap`

### GUSMIDI_ControlChange / GUSMIDI_PitchBend
- Signature: `void GUSMIDI_ControlChange(int channel, int number, int value)` / `void GUSMIDI_PitchBend(int channel, int lsb, int msb)`
- Purpose: Modulate MIDI parameters (CC: volume, expression, sustain, etc.; pitch: frequency offset)
- Inputs: Channel, CC number / pitch LSB & MSB
- Outputs/Return: None
- Side effects: Adjusts active voice parameters (filter, volume, pitch)
- Calls: Not inferable from this file

### GUSMIDI_LoadPatch / GUSMIDI_UnloadPatch
- Signature: `int GUSMIDI_LoadPatch(int prog)` / `int GUSMIDI_UnloadPatch(int prog)`
- Purpose: Load or unload an instrument patch into/from GUS DRAM
- Inputs: Program (instrument) number
- Outputs/Return: Error code
- Side effects: Allocates/frees GUS DRAM; may perform disk I/O (via patch map)
- Calls: Not inferable from this file
- Notes: `GUS_GetPatchMap` resolves program number to filename

### GUSMIDI_SetVolume / GUSMIDI_GetVolume
- Signature: `void GUSMIDI_SetVolume(int volume)` / `int GUSMIDI_GetVolume(void)`
- Purpose: Set or query master output volume
- Inputs: Volume level (0–127 or 0–100; convention not specified)
- Outputs/Return: Current volume
- Side effects: Attenuates all active voices
- Calls: Not inferable from this file

### GUS_ErrorString
- Signature: `char *GUS_ErrorString(int ErrorNumber)`
- Purpose: Translate error code to descriptive string
- Inputs: Error number from `GUS_Errors` enum
- Outputs/Return: Pointer to static error message string
- Side effects: None
- Calls: Not inferable from this file
- Notes: Pointer lifetime tied to static storage; caller must not free

### GUS_GetPatchMap
- Signature: `int GUS_GetPatchMap(char *name)`
- Purpose: Look up patch filename by instrument name
- Inputs: Instrument name string
- Outputs/Return: Program number or error code
- Side effects: Not inferable from this file
- Calls: Not inferable from this file

### D32DosMemAlloc
- Signature: `void *D32DosMemAlloc(unsigned size)`
- Purpose: Allocate DOS-addressable memory for GUS operations
- Inputs: Size in bytes
- Outputs/Return: Pointer to allocated memory (or NULL on failure)
- Side effects: Allocates from real-mode memory pool (GUS compatibility layer)
- Calls: Not inferable from this file
- Notes: Related to 32-bit DOS extender integration

## Control Flow Notes
Expected initialization sequence: `GUS_Init` → `GUSMIDI_Init` → load patches → MIDI events → `GUSMIDI_Shutdown` → `GUS_Shutdown`. Patch loading typically deferred until first `GUSMIDI_ProgramChange` per channel. Not inferable whether MIDI events run in interrupt context or polled loop.

## External Dependencies
- Standard C library (implied: `char *`, `int` pointer operations)
- Not inferable: DOS extender headers, GUS hardware interface headers, patch file format definitions
