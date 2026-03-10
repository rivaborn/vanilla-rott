# audiolib/source/pas16.h

## File Purpose
Public header for the PAS16 audio driver implementation. Declares the interface for controlling a ProAudio Spectrum 16 sound card, including buffered playback/recording, volume control, and driver lifecycle management. Intended for games and audio applications running on legacy DOS/Windows 9x systems with PAS16 hardware.

## Core Responsibilities
- Define error codes for all PAS16 operations
- Expose configuration macros (sample rates, mix modes, DMA, IRQ limits)
- Declare buffered playback and recording functions with callback support
- Provide volume control for PCM and FM synthesis
- Manage audio format negotiation (sampling rate, channels, bit depth)
- Support driver initialization, shutdown, and memory locking for real-time safety

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `PAS_ERRORS` | enum | Error codes and status returns for all PAS16 operations (ranges from `PAS_Warning` = -2 to `PAS_OutOfMemory`) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PAS_DMAChannel` | `unsigned int` | extern | Selected DMA channel for audio transfers |

## Key Functions / Methods

### PAS_Init
- Signature: `int PAS_Init( void )`
- Purpose: Initialize the PAS16 driver and detect hardware
- Inputs: None
- Outputs/Return: Error code from `PAS_ERRORS` enum
- Side effects: Detects card, configures IRQ/DMA, allocates memory
- Calls: Not visible in header
- Notes: Must be called before any other PAS16 function

### PAS_Shutdown
- Signature: `void PAS_Shutdown( void )`
- Purpose: Disable the driver and release all resources
- Inputs: None
- Outputs/Return: None
- Side effects: Stops playback/recording, deallocates memory, restores system state
- Calls: Not visible in header
- Notes: Pairs with `PAS_Init`

### PAS_BeginBufferedPlayback
- Signature: `int PAS_BeginBufferedPlayback( char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void ( *CallBackFunc )( void ) )`
- Purpose: Start buffered audio playback with periodic callback notifications
- Inputs: Buffer pointer, size in bytes, number of divisions (ring buffer sections), sample rate (Hz), mix mode (e.g., `MONO_8BIT`), callback function
- Outputs/Return: Error code from `PAS_ERRORS` enum
- Side effects: Starts DMA transfers, sets up interrupt handler, begins playback
- Calls: Not visible in header
- Notes: Uses ring-buffer model with `NumDivisions` for latency control; callback fires when each division completes

### PAS_BeginBufferedRecord
- Signature: `int PAS_BeginBufferedRecord( char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void ( *CallBackFunc )( void ) )`
- Purpose: Start buffered audio recording with periodic callback notifications
- Inputs: Buffer pointer, size in bytes, number of divisions, sample rate, mix mode, callback function
- Outputs/Return: Error code from `PAS_ERRORS` enum
- Side effects: Starts DMA input, sets up interrupt handler, begins recording
- Calls: Not visible in header
- Notes: Mirrors playback; callback indicates when each buffer division is filled

### PAS_StopPlayback
- Signature: `void PAS_StopPlayback( void )`
- Purpose: Halt audio playback immediately
- Inputs: None
- Outputs/Return: None
- Side effects: Stops DMA, disables interrupt, clears hardware FIFO
- Calls: Not visible in header

### PAS_GetCurrentPos
- Signature: `int PAS_GetCurrentPos( void )`
- Purpose: Query the current playback/record position within the buffer
- Inputs: None
- Outputs/Return: Byte offset into buffer
- Side effects: Reads hardware register
- Calls: Not visible in header
- Notes: Used for synchronization in real-time playback

### PAS_SetPlaybackRate / PAS_GetPlaybackRate
- Signature: `void PAS_SetPlaybackRate( unsigned rate )` / `unsigned PAS_GetPlaybackRate( void )`
- Purpose: Set or query the current sample rate
- Inputs: Sample rate in Hz (valid range `PAS_MinSamplingRate` to `PAS_MaxSamplingRate`: 4000–44000 Hz)
- Outputs/Return: None (set) / current rate in Hz (get)
- Side effects: Configures hardware timer for sample clock
- Calls: Not visible in header

### PAS_SetMixMode
- Signature: `int PAS_SetMixMode( int mode )`
- Purpose: Configure audio format (mono/stereo, 8/16-bit)
- Inputs: Mix mode constant (e.g., `MONO_8BIT`, `STEREO_16BIT`)
- Outputs/Return: Error code
- Side effects: Reconfigures hardware audio format
- Calls: Not visible in header
- Notes: Limited to `PAS_MaxMixMode` (typically `STEREO_16BIT`)

### PAS_SetPCMVolume / PAS_GetPCMVolume
- Signature: `int PAS_SetPCMVolume( int volume )` / `int PAS_GetPCMVolume( void )`
- Purpose: Control digital audio (PCM) volume
- Inputs: Volume level (scale not specified in header)
- Outputs/Return: Error code (set) / current volume (get)
- Side effects: Updates hardware volume register
- Calls: Not visible in header

### PAS_SetFMVolume / PAS_GetFMVolume
- Signature: `void PAS_SetFMVolume( int volume )` / `int PAS_GetFMVolume( void )`
- Purpose: Control FM synthesis volume (separate from PCM)
- Inputs: Volume level
- Outputs/Return: None (set) / current volume (get)
- Side effects: Updates FM volume register
- Calls: Not visible in header

### PAS_GetCardInfo
- Signature: `int PAS_GetCardInfo( int *MaxSampleBits, int *MaxChannels )`
- Purpose: Query hardware capabilities
- Inputs: Pointers to output variables
- Outputs/Return: Fills `MaxSampleBits` and `MaxChannels` with card limits; returns error code
- Side effects: None
- Calls: Not visible in header

### PAS_LockMemory / PAS_UnlockMemory
- Signature: `int PAS_LockMemory( void )` / `void PAS_UnlockMemory( void )`
- Purpose: Lock/unlock memory to prevent paging during DMA operations
- Inputs: None
- Outputs/Return: Error code (lock) / None (unlock)
- Side effects: Calls DPMI/DOS extender to lock physical memory
- Calls: Not visible in header
- Notes: Essential for DOS4GW environment to prevent page faults during interrupt-driven DMA

**Trivial helpers** summarized in Notes:
- `PAS_ErrorString` — maps error codes to human-readable strings
- `PAS_SetCallBack` — sets a general callback function (purpose not clear from signature alone)
- `PAS_SaveMusicVolume` / `PAS_RestoreMusicVolume` — volume state snapshot/restore

## Control Flow Notes
This is a **driver initialization and configuration interface**. Typical usage:
1. `PAS_Init()` — detect and initialize hardware (initialization phase)
2. `PAS_SetMixMode()`, `PAS_SetPlaybackRate()` — configure audio format (configuration phase)
3. `PAS_BeginBufferedPlayback()` — start playback loop with DMA and interrupt callback (frame/update phase, hardware-driven)
4. Per-frame: `PAS_GetCurrentPos()`, volume adjustments via `PAS_SetPCMVolume()`
5. `PAS_StopPlayback()` — halt playback (shutdown phase)
6. `PAS_Shutdown()` — release resources (shutdown phase)

## External Dependencies
- References `STEREO_16BIT` and `MONO_8BIT` constants (defined elsewhere, likely in a mixing/audio library header)
- Implicitly depends on DOS/Windows extended memory and DMA infrastructure
- No explicit `#include` directives in this file
