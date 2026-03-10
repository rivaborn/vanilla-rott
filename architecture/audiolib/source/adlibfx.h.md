# audiolib/source/adlibfx.h

## File Purpose
Public header for the ADLIBFX sound effects module. Defines the interface for Adlib sound card operations including voice management, playback control, volume management, and event callbacks for the legacy audio subsystem.

## Core Responsibilities
- Define error codes for Adlib operations (enum `ADLIBFX_Errors`)
- Define sound effect data structure (`ALSound`)
- Declare voice/sound lifecycle functions (Play, Stop, SoundPlaying)
- Declare voice availability and allocation APIs
- Declare volume control (per-sound and global)
- Declare callback registration for sound completion events
- Declare initialization and shutdown for the Adlib subsystem
- Declare memory locking primitives for DMA/hardware access

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ADLIBFX_Errors` | enum | Error codes returned by API functions |
| `ALSound` | struct | Sound effect data: FM operator parameters, voice assignment, sound data buffer |

## Global / File-Static State
None.

## Key Functions / Methods

### ADLIBFX_Init
- Signature: `int ADLIBFX_Init(void)`
- Purpose: Initialize the Adlib sound subsystem
- Inputs: None
- Outputs/Return: Error code (`ADLIBFX_Errors`)
- Side effects: Initializes hardware/driver state
- Calls: (Implementation in .c file)
- Notes: Must be called before any other ADLIBFX functions

### ADLIBFX_Shutdown
- Signature: `int ADLIBFX_Shutdown(void)`
- Purpose: Shut down the Adlib subsystem and free resources
- Inputs: None
- Outputs/Return: Error code
- Side effects: Halts all playback, deallocates voice state; marked with `#pragma aux ... frame` (fastcall convention)
- Calls: (Implementation in .c file)
- Notes: Should be called at program exit

### ADLIBFX_Play
- Signature: `int ADLIBFX_Play(ALSound *sound, int volume, int priority, unsigned long callbackval)`
- Purpose: Start playback of a sound effect on an available voice
- Inputs: Sound structure, playback volume, voice priority, callback value
- Outputs/Return: Voice handle (≥ `ADLIBFX_MinVoiceHandle` on success, error code on failure)
- Side effects: Allocates a voice, initiates FM synthesis
- Calls: (Implementation in .c file)
- Notes: Returns handle for later Stop/SetVolume operations

### ADLIBFX_Stop
- Signature: `int ADLIBFX_Stop(int handle)`
- Purpose: Stop playback of a sound given its voice handle
- Inputs: Voice handle returned by ADLIBFX_Play
- Outputs/Return: Error code
- Side effects: Releases the voice, stops FM output
- Calls: (Implementation in .c file)

### ADLIBFX_SoundPlaying
- Signature: `int ADLIBFX_SoundPlaying(int handle)`
- Purpose: Query if a sound is currently playing
- Inputs: Voice handle
- Outputs/Return: Status (non-zero if playing, zero if not)
- Side effects: None
- Calls: (Implementation in .c file)

### ADLIBFX_SetVolume
- Signature: `int ADLIBFX_SetVolume(int handle, int volume)`
- Purpose: Adjust playback volume of an active sound
- Inputs: Voice handle, new volume (0–`ADLIBFX_MaxVolume`)
- Outputs/Return: Error code
- Side effects: Updates FM operator levels
- Calls: (Implementation in .c file)

### ADLIBFX_VoiceAvailable
- Signature: `int ADLIBFX_VoiceAvailable(int priority)`
- Purpose: Check if a voice is available at the given priority level
- Inputs: Priority value
- Outputs/Return: Non-zero if available, zero otherwise
- Side effects: None
- Calls: (Implementation in .c file)
- Notes: Used for voice allocation decisions before calling Play

### ADLIBFX_SetCallBack
- Signature: `void ADLIBFX_SetCallBack(void (*function)(unsigned long))`
- Purpose: Register a callback function invoked when sounds complete
- Inputs: Function pointer to callback; receives the `callbackval` passed to ADLIBFX_Play
- Outputs/Return: None
- Side effects: Updates global callback state
- Calls: (Implementation in .c file)
- Notes: Callback likely invoked during interrupt or main loop polling

### ADLIBFX_ErrorString
- Signature: `char *ADLIBFX_ErrorString(int ErrorNumber)`
- Purpose: Convert error code to human-readable message
- Inputs: Error code
- Outputs/Return: String pointer
- Side effects: None
- Calls: (Implementation in .c file)

### PCFX_LockMemory / PCFX_UnlockMemory
- Signature: `int PCFX_LockMemory(void)` / `void PCFX_UnlockMemory(void)`
- Purpose: Lock/unlock code and data in RAM for DMA and interrupt handler access
- Inputs/Outputs: Error code (Lock) / None (Unlock)
- Side effects: Prevents OS from paging memory; critical for ISR and hardware access
- Calls: (Implementation in .c file)
- Notes: Legacy DOS-era pattern; ADLIBFX_UnlockMemory has `#pragma aux ... frame`

## Control Flow Notes
**Initialization:** ADLIBFX_Init must be called at startup; PCFX_LockMemory ensures ISR-safe memory layout.

**Main loop / interrupt:** ADLIBFX_Play queues sounds; playback is driven by interrupt handler or polled frame update (likely ISR-based given callback model). Sounds are tracked by handle; ADLIBFX_SoundPlaying queries live status.

**Cleanup:** ADLIBFX_Shutdown stops all playback; PCFX_UnlockMemory releases DMA lock.

The voice/handle abstraction suggests limited Adlib voices (typically 9 FM channels); priority-based preemption determines which sounds play when over-subscribed.

## External Dependencies
- No includes visible in this header.
- Implementation file (adlibfx.c) likely includes low-level hardware/DPMI APIs.
- `#pragma aux` directives use Watcom C calling convention extensions (frame attribute).
