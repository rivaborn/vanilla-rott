# audiolib/source/pcfx.h

## File Purpose
Public header for a PC sound effects (PCFX) library providing digital audio playback with voice management and priority-based mixing. Defines the engine-facing API for sound initialization, playback control, and shutdown.

## Core Responsibilities
- Define error codes for PCFX operations
- Declare the PCSound data structure for sound effect data
- Declare public API for sound playback and voice management
- Provide volume control and callback mechanism for sound completion
- Define memory locking interface for audio resources

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PCFX_Errors | enum | Error/status codes for PCFX operations |
| PCSound | struct | Sound data container with length, priority, and audio buffer |

## Global / File-Static State
None.

## Key Functions / Methods

### PCFX_Init
- Signature: `int PCFX_Init( void )`
- Purpose: Initialize the PCFX audio system
- Inputs: None
- Outputs/Return: Error code (PCFX_Errors enum)
- Side effects: Initializes audio hardware, allocates voice resources
- Calls: Not inferable from this file
- Notes: Must be called before any PCFX_Play() calls; counterpart is PCFX_Shutdown()

### PCFX_Play
- Signature: `int PCFX_Play( PCSound *sound, int priority, unsigned long callbackval )`
- Purpose: Play a sound effect with priority-based voice allocation
- Inputs: sound (PCSound pointer), priority (voice priority level), callbackval (user callback value)
- Outputs/Return: Voice handle (≥ PCFX_MinVoiceHandle) on success, error code on failure
- Side effects: Allocates audio voice, initiates playback
- Calls: Not inferable from this file
- Notes: Callback triggered via PCFX_SetCallBack(); priority determines if voice is available

### PCFX_Stop
- Signature: `int PCFX_Stop( int handle )`
- Purpose: Stop a currently playing sound
- Inputs: Voice handle (returned by PCFX_Play)
- Outputs/Return: Error code
- Side effects: Deallocates voice resource
- Calls: Not inferable from this file

### PCFX_Shutdown
- Signature: `int PCFX_Shutdown( void )`
- Purpose: Shut down the PCFX audio system
- Inputs: None
- Outputs/Return: Error code
- Side effects: Stops all playback, releases audio hardware and memory
- Calls: Not inferable from this file
- Notes: Pragma aux indicates frame setup; counterpart to PCFX_Init()

### PCFX_SetCallBack
- Signature: `void PCFX_SetCallBack( void ( *function )( unsigned long ) )`
- Purpose: Register a callback function invoked when sounds complete playback
- Inputs: Function pointer (takes unsigned long callbackval, returns void)
- Outputs/Return: None
- Side effects: Updates global callback function pointer
- Calls: Not inferable from this file

## Control Flow Notes
Lifecycle: PCFX_Init() → PCFX_Play()/PCFX_Stop() (multiple times) → PCFX_Shutdown(). Memory locking (PCFX_LockMemory/UnlockMemory) suggests resource management for real-time audio. Volume and lookup table functions allow runtime tuning. Callbacks provide completion notification to caller.

## External Dependencies
- No includes visible (header guard only)
- Implementation referenced as ADLIBFX.C per header comment
- Symbols for audio driver interaction not defined in this file
