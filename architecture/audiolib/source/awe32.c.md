# audiolib/source/awe32.c

## File Purpose
Provides wrapper functions for the AWE32 sound card driver on DOS systems, enabling MIDI playback and hardware control. Serves as a translation layer between the game engine and the AWE32 low-level library, handling initialization, shutdown, and real-time MIDI operations.

## Core Responsibilities
- Detect and initialize AWE32 sound card hardware and MIDI interface
- Provide MIDI control functions (note on/off, pitch bend, program change, aftertouch)
- Manage base I/O addresses for Sound Blaster, EMU8000, and MPU-401
- Lock time-critical code and data into memory for real-time safety via DPMI
- Track active notes per channel for cleanup operations
- Load built-in SoundFont presets and configure banks
- Report errors via centralized error code system

## Key Types / Data Structures
| Name | Kind | Purpose |
| --- | --- | --- |
| SOUND_PACKET | struct | Sound font configuration, bank sizing, patch RAM tracking |
| BLASTER_CONFIG | struct | Hardware I/O addresses and configuration from Sound Blaster |
| dpmi_regs | struct | x86 register state for protected mode interrupts |

## Global / File-Static State
| Name | Type | Scope | Purpose |
| --- | --- | --- | --- |
| wSBCBaseAddx | WORD | static | Sound Blaster card base I/O address |
| wEMUBaseAddx | WORD | static | EMU8000 subsystem base I/O address |
| wMpuBaseAddx | WORD | static | MPU-401 MIDI interface base I/O address |
| NoteFlags | unsigned short[128] | static | Per-key bit mask tracking active channels (1 bit per channel) |
| spSound | SOUND_PACKET | static | Sound packet config for SoundFont loading |
| lBankSizes | LONG[MAXBANKS] | static | Bank size array for sound font management |
| AWE32_ErrorCode | int | global | Current error code; used by error reporting |

## Key Functions / Methods

### AWE32_Init
- Signature: `int AWE32_Init(void)`
- Purpose: Detect, configure, and initialize AWE32 hardware; lock all time-critical code/data
- Inputs: None
- Outputs/Return: `AWE32_Ok` on success; `AWE32_Error` on failure (sets `AWE32_ErrorCode`)
- Side effects: Initializes static base addresses; performs hardware detection; locks entire MIDI code region and data via DPMI
- Calls: `BLASTER_GetCardSettings`, `BLASTER_GetEnv`, `awe32Detect`, `awe32InitHardware`, `awe32InitMIDI`, `awe32TotalPatchRam`, `LoadSBK`, `awe32InitNRPN`, `DPMI_LockMemoryRegion`, `DPMI_Lock`, `DPMI_LockMemory`
- Notes: Falls back to default I/O addresses (0x220, 0x330, 0x620) if not found in BLASTER config or environment; locks multiple code/data regions from AWE32 library for interrupt safety

### AWE32_Shutdown
- Signature: `void AWE32_Shutdown(void)`
- Purpose: Unlock all memory regions and terminate hardware
- Inputs: None
- Outputs/Return: None
- Side effects: Reverses all locks from `AWE32_Init`; resets MPU-401 MIDI interface
- Calls: `ShutdownMPU`, `awe32Terminate`, `DPMI_UnlockMemoryRegion`, `DPMI_Unlock`, `DPMI_UnlockMemory`

### AWE32_NoteOn
- Signature: `void AWE32_NoteOn(int channel, int key, int velocity)`
- Purpose: Start a note on a MIDI channel
- Inputs: Channel (0–15), key (0–127), velocity (0–127)
- Outputs/Return: None
- Side effects: Calls low-level `awe32NoteOn`; sets corresponding bit in `NoteFlags[key]` to track active note
- Calls: `SetES`, `awe32NoteOn`, `RestoreES`
- Notes: Memory-locked function; uses bitwise OR to mark channel as active

### AWE32_NoteOff
- Signature: `void AWE32_NoteOff(int channel, int key, int velocity)`
- Purpose: Stop a note on a MIDI channel
- Inputs: Channel (0–15), key (0–127), velocity (0–127)
- Outputs/Return: None
- Side effects: Calls low-level `awe32NoteOff`; clears corresponding bit in `NoteFlags[key]`
- Calls: `SetES`, `awe32NoteOff`, `RestoreES`
- Notes: Memory-locked function; uses XOR to toggle bit off

### AWE32_ProgramChange
- Signature: `void AWE32_ProgramChange(int channel, int program)`
- Purpose: Change instrument/timbre for a MIDI channel
- Inputs: Channel (0–15), program number (0–127)
- Outputs/Return: None
- Side effects: Updates hardware instrument selection
- Calls: `SetES`, `awe32ProgramChange`, `RestoreES`
- Notes: Memory-locked function

### AWE32_PitchBend
- Signature: `void AWE32_PitchBend(int channel, int lsb, int msb)`
- Purpose: Apply pitch bend to channel
- Inputs: Channel (0–15), pitch bend LSB and MSB components
- Outputs/Return: None
- Side effects: Updates hardware pitch bend
- Calls: `SetES`, `awe32PitchBend`, `RestoreES`
- Notes: Memory-locked function

### AWE32_ControlChange
- Signature: `void AWE32_ControlChange(int channel, int number, int value)`
- Purpose: Send MIDI control change (CC) message
- Inputs: Channel (0–15), CC number, value (0–127)
- Outputs/Return: None
- Side effects: If CC 0x7b (all notes off), iterates `NoteFlags` and turns off all active notes; otherwise calls low-level controller handler
- Calls: `SetES`, `awe32NoteOff`, `awe32Controller`, `RestoreES`
- Notes: Memory-locked function; special handling for all-notes-off to clean up tracked notes

### AWE32_PolyAftertouch
- Signature: `void AWE32_PolyAftertouch(int channel, int key, int pressure)`
- Purpose: Apply polyphonic (per-note) aftertouch pressure
- Inputs: Channel (0–15), key (0–127), pressure (0–127)
- Outputs/Return: None
- Side effects: Updates hardware key pressure
- Calls: `SetES`, `awe32PolyKeyPressure`, `RestoreES`
- Notes: Memory-locked function

### AWE32_ChannelAftertouch
- Signature: `void AWE32_ChannelAftertouch(int channel, int pressure)`
- Purpose: Apply channel-wide aftertouch pressure
- Inputs: Channel (0–15), pressure (0–127)
- Outputs/Return: None
- Side effects: Updates hardware channel pressure
- Calls: `SetES`, `awe32ChannelPressure`, `RestoreES`
- Notes: Memory-locked function

### AWE32_ErrorString
- Signature: `char *AWE32_ErrorString(int ErrorNumber)`
- Purpose: Return human-readable error message for error codes
- Inputs: Error number (or -1 for current `AWE32_ErrorCode`)
- Outputs/Return: Pointer to static error string
- Side effects: May recursively call itself if input is -1 or -2
- Calls: Recursive call; `BLASTER_ErrorString`
- Notes: Delegates Sound Blaster errors to `BLASTER_ErrorString`; handles warning/error indirection

## Control Flow Notes
- **Initialization**: `AWE32_Init` detects hardware via BLASTER config, locks memory regions for interrupt safety, initializes MIDI, and loads SoundFont presets.
- **MIDI playback**: Note on/off functions maintain `NoteFlags` to track active voices; `ControlChange` special-cases all-notes-off (0x7b) to force cleanup.
- **Real-time safety**: All MIDI functions use `SetES`/`RestoreES` inline assembly to save/restore segment registers before calling low-level library.
- **Shutdown**: Reverses memory locks and terminates hardware cleanly.

## External Dependencies
- **Includes**: `conio.h` (I/O), `string.h` (memset), `dpmi.h` (memory locking), `blaster.h` (Sound Blaster), `ctaweapi.h` (AWE32 API)
- **External symbols**: `awe32NoteOn`, `awe32NoteOff`, `awe32ProgramChange`, `awe32Controller`, `awe32PolyKeyPressure`, `awe32ChannelPressure`, `awe32PitchBend`, `awe32Detect`, `awe32InitHardware`, `awe32InitMIDI`, `awe32Terminate`, `awe32InitNRPN`, `awe32TotalPatchRam`, `awe32DefineBankSizes`, `awe32SoundPad`, `awe32SPadXObj` (1–7), `awe32NumG`, `__midieng_code`, `__midieng_ecode`, `__nrpn_code`, `__nrpn_ecode`, `__midivar_data`, `__nrpnvar_data`, `__embed_data`, `BLASTER_GetCardSettings`, `BLASTER_GetEnv`, `BLASTER_ErrorString`, `DPMI_LockMemoryRegion`, `DPMI_Lock`, `DPMI_Unlock`, `DPMI_UnlockMemoryRegion`, `DPMI_UnlockMemory`, `DPMI_LockMemory`, `inp`, `outp`
