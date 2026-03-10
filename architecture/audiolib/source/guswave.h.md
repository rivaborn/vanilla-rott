# audiolib/source/guswave.h

## File Purpose
Public API header for the GUS (Gravis UltraSound) Wave audio module. Defines the interface for playing sampled audio (WAV/VOC) files through GUS hardware, supporting voice management, 3D audio positioning, and real-time playback control.

## Core Responsibilities
- Voice allocation and lifecycle management (init, play, kill, shutdown)
- WAV and VOC file playback with pitch/pan control
- Demand-feed (callback-driven) playback for dynamic audio generation
- Volume control and stereo reversal configuration
- Error reporting for GUS hardware and audio operations
- Voice activity tracking and availability queries

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GUSWAVE_Errors` | enum | Error codes for GUS operations (GUSError, NotInstalled, InvalidVOCFile, InvalidWAVFile, etc.) |

## Global / File-Static State
None.

## Key Functions / Methods

### GUSWAVE_Init
- Signature: `int GUSWAVE_Init(int numvoices)`
- Purpose: Initialize the GUS audio module with a specified number of concurrent voices
- Inputs: `numvoices` – number of concurrent audio voices to allocate
- Outputs/Return: Error code (enum GUSWAVE_Errors)
- Side effects: Initializes GUS hardware and internal voice state
- Calls: Not inferable from header
- Notes: Must be called before other GUSWAVE_* functions

### GUSWAVE_Shutdown
- Signature: `void GUSWAVE_Shutdown(void)`
- Purpose: Clean shutdown of GUS audio module and hardware
- Inputs: None
- Outputs/Return: Void
- Side effects: Stops all audio playback, deallocates voice resources, releases GUS hardware
- Calls: Not inferable from header
- Notes: `#pragma aux GUSWAVE_Shutdown frame;` indicates Watcom C ABI annotation for frame setup

### GUSWAVE_PlayVOC
- Signature: `int GUSWAVE_PlayVOC(char *sample, int pitchoffset, int angle, int volume, int priority, unsigned long callbackval)`
- Purpose: Play a VOC (Creative Voice) audio sample through an available voice
- Inputs: `sample` (VOC data pointer), `pitchoffset` (pitch adjustment), `angle` (3D pan/direction 0–359°), `volume` (playback level), `priority` (voice allocation priority), `callbackval` (callback identifier)
- Outputs/Return: Voice handle (≥ GUSWAVE_MinVoiceHandle) or error code
- Side effects: Allocates a voice and begins playback; invokes callback on completion
- Calls: Not inferable from header
- Notes: Handle returned is used with GUSWAVE_Kill, GUSWAVE_SetPitch, GUSWAVE_SetPan3D

### GUSWAVE_PlayWAV
- Signature: `int GUSWAVE_PlayWAV(char *sample, int pitchoffset, int angle, int volume, int priority, unsigned long callbackval)`
- Purpose: Play a WAV (RIFF WAVE) audio sample through an available voice
- Inputs: Same as GUSWAVE_PlayVOC
- Outputs/Return: Voice handle or error code
- Side effects: Allocates a voice and begins playback; invokes callback on completion
- Calls: Not inferable from header
- Notes: Parallel API to GUSWAVE_PlayVOC for WAV format support

### GUSWAVE_StartDemandFeedPlayback
- Signature: `int GUSWAVE_StartDemandFeedPlayback(void (*function)(char **ptr, unsigned long *length), int channels, int bits, int rate, int pitchoffset, int angle, int volume, int priority, unsigned long callbackval)`
- Purpose: Start real-time audio generation via callback (demand-feed) rather than pre-loaded sample
- Inputs: `function` (callback to fetch audio chunks), `channels` (mono/stereo), `bits` (sample bit depth), `rate` (sample rate Hz), pitch/angle/volume/priority/callbackval (as above)
- Outputs/Return: Voice handle or error code
- Side effects: Allocates a voice and repeatedly calls `function` to pull audio data
- Calls: Not inferable from header
- Notes: Callback function updates `ptr` (audio buffer pointer) and `length` (bytes available)

### GUSWAVE_Kill
- Signature: `int GUSWAVE_Kill(int handle)`
- Purpose: Stop and free a specific voice by handle
- Inputs: `handle` – voice handle from GUSWAVE_Play* or GUSWAVE_StartDemandFeedPlayback
- Outputs/Return: Error code
- Side effects: Stops audio playback; deallocates the voice; may trigger completion callback
- Calls: Not inferable from header

### GUSWAVE_KillAllVoices
- Signature: `int GUSWAVE_KillAllVoices(void)`
- Purpose: Stop all active voices immediately
- Inputs: None
- Outputs/Return: Error code
- Side effects: Halts all audio playback
- Calls: Not inferable from header

### GUSWAVE_SetPitch
- Signature: `int GUSWAVE_SetPitch(int handle, int pitchoffset)`
- Purpose: Adjust the pitch of a playing voice post-start
- Inputs: `handle` (voice), `pitchoffset` (pitch shift value)
- Outputs/Return: Error code
- Side effects: Modifies voice playback pitch without stopping
- Calls: Not inferable from header

### GUSWAVE_SetPan3D
- Signature: `int GUSWAVE_SetPan3D(int handle, int angle, int distance)`
- Purpose: Update 3D spatial positioning (azimuth and depth) for a voice
- Inputs: `handle` (voice), `angle` (azimuth 0–359°), `distance` (attenuation/distance cue)
- Outputs/Return: Error code
- Side effects: Updates stereo pan and volume based on 3D position
- Calls: Not inferable from header

### GUSWAVE_SetVolume
- Signature: `void GUSWAVE_SetVolume(int volume)`
- Purpose: Set global master volume for all GUS audio output
- Inputs: `volume` (master level)
- Outputs/Return: Void
- Side effects: Affects all active voices
- Calls: Not inferable from header

### GUSWAVE_VoicePlaying
- Signature: `int GUSWAVE_VoicePlaying(int handle)`
- Purpose: Query whether a specific voice is still actively playing
- Inputs: `handle` (voice)
- Outputs/Return: Boolean-like (non-zero = playing, zero = stopped/invalid)
- Side effects: None
- Calls: Not inferable from header

### GUSWAVE_VoicesPlaying
- Signature: `int GUSWAVE_VoicesPlaying(void)`
- Purpose: Get the count of currently active voices
- Inputs: None
- Outputs/Return: Integer count of playing voices
- Side effects: None
- Calls: Not inferable from header

### GUSWAVE_VoiceAvailable
- Signature: `int GUSWAVE_VoiceAvailable(int priority)`
- Purpose: Check if a voice can be allocated for a given priority level
- Inputs: `priority` (requested priority)
- Outputs/Return: Non-zero if available; may trigger voice preemption logic based on priority
- Side effects: None (query only)
- Calls: Not inferable from header

### GUSWAVE_SetCallBack
- Signature: `void GUSWAVE_SetCallBack(void (*function)(unsigned long))`
- Purpose: Register a global callback function invoked when voices complete playback
- Inputs: `function` (completion callback; receives `callbackval` from voice launch)
- Outputs/Return: Void
- Side effects: Installs interrupt-level or deferred callback handler
- Calls: Not inferable from header

### GUSWAVE_ErrorString
- Signature: `char *GUSWAVE_ErrorString(int ErrorNumber)`
- Purpose: Return human-readable error message for a given error code
- Inputs: `ErrorNumber` (GUSWAVE_Errors enum value)
- Outputs/Return: Pointer to error description string
- Side effects: None
- Calls: Not inferable from header

### GUSWAVE_SetReverseStereo / GUSWAVE_GetReverseStereo / GUSWAVE_GetVolume
- Purpose: Utility functions for stereo configuration and volume query
- Inputs: `setting` (GUSWAVE_SetReverseStereo), none (GUSWAVE_GetReverseStereo/GetVolume)
- Outputs/Return: Void or int (status/level)
- Side effects: GUSWAVE_SetReverseStereo modifies stereo channel order
- Notes: Summarized; minimal logic

## Control Flow Notes
**Init/Shutdown pattern**: Module is initialized once with GUSWAVE_Init (allocating voice pool), used for playback via Play* functions, and cleaned with GUSWAVE_Shutdown.

**Playback lifecycle**: PlayVOC/PlayWAV/StartDemandFeedPlayback allocate a voice, return a handle, and fire a completion callback when done (or when killed via GUSWAVE_Kill). Real-time parameters (pitch, pan) can be modified mid-playback.

**Voice preemption**: GUSWAVE_VoiceAvailable suggests priority-based voice stealing if all voices are in use.

**Interrupt context**: The callback mechanism (SetCallBack) likely runs at hardware interrupt level (ISR), typical of 1990s sound card drivers.

## External Dependencies
- `stdio.h` (implied by char pointer and error string API)
- GUS hardware driver layer (not inferable; implemented in guswave.c)
- Watcom C compiler (pragma aux directives)
