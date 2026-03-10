# audiolib/source/blaster.h

## File Purpose
Public header for the BLASTER audio library, providing a C interface to Sound Blaster compatible audio cards. Defines configuration structures, error codes, card types, and function declarations for audio playback, recording, DSP control, and mixer operations on legacy Sound Blaster hardware.

## Core Responsibilities
- Define Sound Blaster hardware configuration and capability structures
- Declare error codes, card types, and audio format constants
- Provide DSP (Digital Signal Processor) read/write and reset operations
- Manage audio playback and recording with buffering and DMA
- Control mixer hardware (volume, speaker on/off)
- Handle interrupt and DMA channel configuration
- Provide callback mechanisms for audio completion events
- Lock/unlock memory for DMA-safe operations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| BLASTER_CONFIG | struct | Hardware configuration: I/O address, card type, IRQ, DMA channels, MIDI/Emu settings |
| BLASTER_ERRORS | enum | Error codes: success, warnings, configuration/hardware errors, memory/DPMI errors |
| BLASTER_Types | enum | Sound Blaster card variants: SB, SBPro, SB20, SBPro2, SB16 |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| BLASTER_Config | BLASTER_CONFIG | extern | Current Sound Blaster hardware configuration |
| BLASTER_DMAChannel | int | extern | Active DMA channel in use |

## Key Functions / Methods

### BLASTER_Init / BLASTER_Shutdown
- Signature: `int BLASTER_Init(void)` / `void BLASTER_Shutdown(void)`
- Purpose: Initialize Sound Blaster hardware and driver state; clean up resources
- Inputs: None
- Outputs/Return: BLASTER_Init returns error code; Shutdown returns void
- Side effects: Configures hardware, allocates memory, registers interrupts
- Calls: (implementation in blaster.c)
- Notes: Shutdown must be called to restore system state

### BLASTER_BeginBufferedPlayback / BLASTER_BeginBufferedRecord
- Signature: `int BLASTER_BeginBufferedPlayback(char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void (*CallBackFunc)(void))`
- Purpose: Start audio playback or recording with DMA-driven circular buffering
- Inputs: Audio buffer pointer, size, division count, sample rate, mix mode (mono/stereo, 8/16-bit), callback function pointer
- Outputs/Return: Error code
- Side effects: Sets up DMA transfers, enables interrupts, configures DSP playback/record mode
- Calls: (implementation in blaster.c)
- Notes: Callback invoked when each division of buffer is processed

### BLASTER_SetupDMABuffer
- Signature: `int BLASTER_SetupDMABuffer(char *BufferPtr, int BufferSize, int mode)`
- Purpose: Configure DMA buffer for audio transfer
- Inputs: Buffer pointer, size in bytes, mix mode flags
- Outputs/Return: Error code
- Side effects: Locks memory, sets DMA controller registers
- Notes: Must be called before playback/record operations

### BLASTER_WriteDSP / BLASTER_ReadDSP / BLASTER_ResetDSP
- Signature: `int BLASTER_WriteDSP(unsigned data)` / `int BLASTER_ReadDSP(void)` / `int BLASTER_ResetDSP(void)`
- Purpose: Low-level DSP communication—write/read DSP registers, reset DSP to known state
- Inputs: WritesDSP takes data byte; others take none
- Outputs/Return: ReadDSP returns DSP data; others return error codes
- Side effects: Hardware register I/O
- Notes: Blocks until operation completes; used for audio format and mode configuration

### BLASTER_SetPlaybackRate / BLASTER_GetPlaybackRate
- Signature: `void BLASTER_SetPlaybackRate(unsigned rate)` / `unsigned BLASTER_GetPlaybackRate(void)`
- Purpose: Set or query the audio sample rate
- Inputs: Sample rate in Hz (e.g., 11000, 22000, 44000)
- Outputs/Return: GetPlaybackRate returns current rate in Hz
- Side effects: Configures DSP timing registers
- Notes: Must be set before starting playback

### BLASTER_SetMixMode
- Signature: `int BLASTER_SetMixMode(int mode)`
- Purpose: Configure audio format (mono/stereo, 8-bit/16-bit)
- Inputs: Mode flags: MONO_8BIT, STEREO_8BIT, MONO_16BIT, STEREO_16BIT
- Outputs/Return: Error code
- Side effects: Configures DSP audio format
- Notes: SetMixMode at initialization; macro constants provided for standard modes

### BLASTER_WriteMixer / BLASTER_ReadMixer
- Signature: `void BLASTER_WriteMixer(int reg, int data)` / `int BLASTER_ReadMixer(int reg)`
- Purpose: Low-level mixer register control
- Inputs: Register address, data (write only)
- Outputs/Return: ReadMixer returns register value
- Side effects: Hardware mixer I/O
- Calls: (implementation in blaster.c)
- Notes: Used internally by volume and speaker control functions

### BLASTER_GetVoiceVolume / BLASTER_SetVoiceVolume / BLASTER_GetMidiVolume / BLASTER_SetMidiVolume
- Signature: `int BLASTER_GetVoiceVolume(void)` / `int BLASTER_SetVoiceVolume(int volume)` / (MIDI variants)
- Purpose: Query or set audio output volume for PCM and MIDI channels
- Inputs: Volume level (0–15 typical range)
- Outputs/Return: Get functions return current volume; Set returns error code
- Side effects: Writes mixer registers
- Notes: SaveVoiceVolume/RestoreVoiceVolume pair for state preservation

### BLASTER_GetEnv / BLASTER_SetCardSettings / BLASTER_GetCardSettings
- Signature: `int BLASTER_GetEnv(BLASTER_CONFIG *Config)` / `int BLASTER_SetCardSettings(BLASTER_CONFIG Config)` / `int BLASTER_GetCardSettings(BLASTER_CONFIG *Config)`
- Purpose: Load hardware configuration from BLASTER environment variable; apply or retrieve card configuration
- Inputs: Config pointer or structure
- Outputs/Return: Error codes; GetCardSettings and GetEnv fill output structure
- Side effects: GetEnv parses environment; SetCardSettings reconfigures hardware
- Notes: Config must include Address, Type, Interrupt, Dma8, Dma16 fields

### BLASTER_SetCallBack
- Signature: `void BLASTER_SetCallBack(void (*func)(void))`
- Purpose: Register a callback function invoked on audio interrupt/buffer division
- Inputs: Function pointer (void → void)
- Side effects: Stores callback for invocation during playback/record
- Notes: Called from interrupt handler during DMA transfers

## Control Flow Notes
**Initialization**: BLASTER_Init → BLASTER_GetEnv or BLASTER_SetCardSettings → BLASTER_SetPlaybackRate → BLASTER_SetMixMode → BLASTER_LockMemory

**Playback/Record**: BLASTER_SetupDMABuffer → BLASTER_BeginBufferedPlayback/Record (with callback) → [interrupt triggers callback at buffer divisions] → BLASTER_StopPlayback → BLASTER_Shutdown

**Mixer/Volume**: Queries and adjustments via mixer register APIs; Save/Restore pairs allow state preservation across operations.

## External Dependencies
- None visible in this header (standard C types only)
- Implementation (blaster.c) likely includes: ISA hardware port I/O, DOS/DPMI memory locking, interrupt handling
