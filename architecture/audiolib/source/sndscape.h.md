# audiolib/source/sndscape.h

## File Purpose
Public header for the Soundscape audio driver module. Declares the interface for initializing, configuring, and controlling a Soundscape sound card for buffered PCM playback with DMA and interrupt-driven callbacks.

## Core Responsibilities
- Audio device initialization and hardware detection (Soundscape-specific)
- Playback rate and mixing mode configuration
- Buffered playback management with DMA transfers
- Callback-driven audio completion events
- Error reporting and hardware capability queries
- MIDI port and IRQ configuration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SOUNDSCAPE_ERRORS` | enum | Error codes for audio operations (warnings, missing configs, hardware errors, memory issues) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SOUNDSCAPE_DMAChannel` | int | global | The DMA channel in use by the Soundscape driver |
| `SOUNDSCAPE_ErrorCode` | int | global | Last error code from a Soundscape operation |

## Key Functions / Methods

### SOUNDSCAPE_Init
- Signature: `int SOUNDSCAPE_Init(void)`
- Purpose: Initialize the Soundscape audio hardware; detect card, load driver, configure IRQ/DMA
- Inputs: None
- Outputs/Return: Error code (SOUNDSCAPE_ERRORS enum value)
- Side effects: Sets `SOUNDSCAPE_DMAChannel`, `SOUNDSCAPE_ErrorCode`; initializes hardware
- Calls: (Not visible in header)
- Notes: Must be called before any other Soundscape functions; checks environment variables and initialization files

### SOUNDSCAPE_Shutdown
- Signature: `void SOUNDSCAPE_Shutdown(void)`
- Purpose: Shut down the Soundscape driver and release hardware resources
- Inputs: None
- Outputs/Return: None
- Side effects: Releases DMA channel, IRQ, hardware state
- Calls: (Not visible in header)
- Notes: Should be called during engine shutdown

### SOUNDSCAPE_BeginBufferedPlayback
- Signature: `int SOUNDSCAPE_BeginBufferedPlayback(char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void (*CallBackFunc)(void))`
- Purpose: Start playing audio from a DMA buffer; divide buffer into sections with callback on completion of each
- Inputs: Audio buffer pointer, total buffer size, number of divisions, sample rate (Hz), mix mode, callback function pointer
- Outputs/Return: Error code
- Side effects: Initiates DMA transfers; registers interrupt handler; starts playback
- Calls: (Not visible in header)
- Notes: Buffer must be DMA-accessible; callback fires when each division completes

### SOUNDSCAPE_SetPlaybackRate
- Signature: `void SOUNDSCAPE_SetPlaybackRate(unsigned rate)`
- Purpose: Set the playback sample rate
- Inputs: Sample rate in Hz
- Outputs/Return: None
- Side effects: Changes active playback rate
- Calls: (Not visible in header)

### SOUNDSCAPE_GetPlaybackRate
- Signature: `unsigned SOUNDSCAPE_GetPlaybackRate(void)`
- Purpose: Retrieve the current playback sample rate
- Inputs: None
- Outputs/Return: Current sample rate in Hz
- Side effects: None

### SOUNDSCAPE_SetMixMode
- Signature: `int SOUNDSCAPE_SetMixMode(int mode)`
- Purpose: Configure the audio mix mode (mono/stereo, bit depth, etc.)
- Inputs: Mix mode value
- Outputs/Return: Error code
- Side effects: Changes audio format configuration
- Calls: (Not visible in header)

### SOUNDSCAPE_StopPlayback
- Signature: `void SOUNDSCAPE_StopPlayback(void)`
- Purpose: Stop DMA playback and silence the device
- Inputs: None
- Outputs/Return: None
- Side effects: Halts DMA transfers; disables audio output
- Calls: (Not visible in header)

### SOUNDSCAPE_GetCurrentPos
- Signature: `int SOUNDSCAPE_GetCurrentPos(void)`
- Purpose: Query the current playback position within the buffer
- Inputs: None
- Outputs/Return: Current position (likely sample offset or division index)
- Side effects: None

### SOUNDSCAPE_SetCallBack
- Signature: `void SOUNDSCAPE_SetCallBack(void (*func)(void))`
- Purpose: Register or update the callback function for playback events
- Inputs: Function pointer to callback
- Outputs/Return: None
- Side effects: Updates interrupt handler callback
- Calls: (Not visible in header)

### SOUNDSCAPE_ErrorString
- Signature: `char *SOUNDSCAPE_ErrorString(int ErrorNumber)`
- Purpose: Convert error code to human-readable string
- Inputs: Error code from SOUNDSCAPE_ERRORS enum
- Outputs/Return: Pointer to error message string
- Side effects: None

### SOUNDSCAPE_GetCardInfo
- Signature: `int SOUNDSCAPE_GetCardInfo(int *MaxSampleBits, int *MaxChannels)`
- Purpose: Query hardware capabilities (bit depth, channel count)
- Inputs: Pointers to output variables for sample bits and channel count
- Outputs/Return: Error code; output parameters filled with hardware specs
- Side effects: None

### SOUNDSCAPE_GetMIDIPort
- Signature: `int SOUNDSCAPE_GetMIDIPort(void)`
- Purpose: Get the I/O port address for MIDI communication
- Inputs: None
- Outputs/Return: MIDI port address
- Side effects: None

## Control Flow Notes
Typical initialization and playback cycle: `SOUNDSCAPE_Init()` → `SOUNDSCAPE_BeginBufferedPlayback()` with callback registered → playback proceeds with interrupts firing callbacks on buffer divisions → `SOUNDSCAPE_StopPlayback()` or callback loop → `SOUNDSCAPE_Shutdown()` on engine exit. Fits into the engine's audio subsystem initialization (pre-frame) and runs asynchronously via DMA/IRQ during the main game loop.

## External Dependencies
- None visible in header (implementation in sndscape.c likely includes hardware I/O, DMA setup, IRQ handler registration)
- Assumes DOS/x86 real-mode or protected-mode (DPMI) environment with ISA hardware access
