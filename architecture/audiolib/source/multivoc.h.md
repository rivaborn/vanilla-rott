# audiolib/source/multivoc.h

## File Purpose
Public API header for MULTIVOC, a multi-voice software audio mixer library. Defines the interface for initializing the audio system, managing voice channels, controlling playback parameters, and playing various audio formats (raw, WAV, VOC).

## Core Responsibilities
- Define error codes and global error state for audio operations
- Declare voice lifecycle functions (allocate, play, kill)
- Declare playback control functions (pitch, pan, reverb, 3D audio)
- Declare audio format loaders (WAV, VOC, raw data)
- Declare mixing mode and volume configuration
- Declare system initialization and memory management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MV_Errors | enum | Error/status codes (Warning, Error, Ok, UnsupportedCard, NoVoices, InvalidWAVFile, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MV_ErrorCode | int | global extern | Current error code from last MULTIVOC operation |

## Key Functions / Methods

### MV_Init
- Signature: `int MV_Init(int soundcard, int MixRate, int Voices, int numchannels, int samplebits);`
- Purpose: Initialize audio system with hardware and mixing parameters
- Inputs: soundcard ID, mixing sample rate, voice count, channel count (mono/stereo), sample bits (8/16)
- Outputs/Return: Error code (MV_Errors enum value)
- Side effects: Allocates voice buffers, configures hardware, sets global state
- Notes: Must be called before any playback; critical initialization function

### MV_Shutdown
- Signature: `int MV_Shutdown(void);`
- Purpose: Shut down audio system and free resources
- Inputs: None
- Outputs/Return: Error code
- Side effects: Stops all voices, deallocates buffers, disables hardware

### MV_PlayRaw
- Signature: `int MV_PlayRaw(char *ptr, unsigned long length, unsigned rate, int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval);`
- Purpose: Play raw audio data from memory buffer
- Inputs: Audio buffer pointer, length, sample rate, pitch offset, volume, stereo balance (left/right), priority, callback identifier
- Outputs/Return: Voice handle (≥ MV_MinVoiceHandle) or error code
- Side effects: Allocates voice channel, begins playback, may invoke callbacks on completion
- Notes: Voice handle used with MV_Kill, MV_SetPitch, MV_SetPan for runtime control

### MV_PlayWAV / MV_PlayVOC
- Similar signatures to MV_PlayRaw but accept WAV/VOC file pointers; library handles format parsing
- Return voice handles for further control
- Error codes: MV_InvalidWAVFile, MV_InvalidVOCFile for malformed data

### MV_PlayWAV3D / MV_PlayVOC3D
- Variants accepting 3D positioning (angle, distance) instead of stereo panning
- Internally compute left/right channels from spatial parameters

### MV_SetPan / MV_Pan3D
- Modify stereo panning or 3D position of active voice
- MV_Pan3D abstracts angle/distance to stereo parameters

**Other notable functions** (listed briefly): MV_SetPitch, MV_SetFrequency, MV_EndLooping, MV_SetReverb, MV_SetVolume, MV_VoiceAvailable, MV_Kill, MV_KillAllVoices, MV_StartRecording, MV_StartDemandFeedPlayback.

## Control Flow Notes
- **Init phase**: MV_Init called at startup with hardware parameters
- **Playback phase**: During frame updates, MV_Play* functions called to start sounds; MV_Set* functions adjust active voices; MV_Kill stops voices
- **Recording**: MV_StartRecording/StopRecord run parallel to playback
- **Shutdown**: MV_Shutdown on exit
- MV_StartPlayback/StopPlayback control when mixing engine produces output

## External Dependencies
- No standard library includes visible
- Callback functions supplied as function pointers (`void (*function)(...)`)
- Hardware abstraction via soundcard parameter in MV_Init
- All implementation details defined elsewhere (MULTIVOC.C)
