# audiolib/public/include/fx_man.h

## File Purpose
Public API header for the FX_MAN sound effects management module. Provides high-level sound initialization, playback control, and mixing for the Apogee audiolib, supporting multiple sound card types and audio formats (VOC, WAV, raw PCM) with 3D positioning and effects.

## Core Responsibilities
- Initialize and configure sound cards (Sound Blaster variants, Pro Audio Spectrum, Adlib, etc.)
- Manage voice allocation and priority-based sound queuing
- Play audio samples in multiple formats with pitch, panning, and volume control
- Support looped and non-looped playback with callback notifications
- Implement 3D audio positioning (angle/distance)
- Provide reverb and spatial audio effects
- Handle audio recording and demand-feed playback
- Manage global mixer settings (volume, stereo reverse, reverb depth/delay)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `fx_device` | struct | Holds sound card capabilities: max voices, sample bits, channels |
| `fx_blaster_config` | struct | Sound Blaster hardware configuration (address, interrupt, DMA, MIDI, EMU settings) |
| `FX_ERRORS` | enum | Error codes (FX_Ok, FX_ASSVersion, FX_BlasterError, FX_SoundCardError, etc.) |
| `fx_BLASTER_Types` | enum | Sound Blaster variants (SB, SBPro, SB20, SBPro2, SB16) |

## Global / File-Static State
None.

## Key Functions / Methods

### FX_Init
- Signature: `int FX_Init( int SoundCard, int numvoices, int numchannels, int samplebits, unsigned mixrate )`
- Purpose: Initialize the sound effects system with specified hardware and mixer parameters
- Inputs: Sound card type (enum), voice count, channel count (mono/stereo), sample resolution, mixing rate
- Outputs/Return: Error code (FX_Ok on success, negative on failure)
- Side effects: Allocates mixer buffers, initializes hardware, sets global audio state

### FX_SetupCard / FX_SetupSoundBlaster
- Signature: `int FX_SetupCard( int SoundCard, fx_device *device )` / `int FX_SetupSoundBlaster( fx_blaster_config blaster, int *MaxVoices, int *MaxSampleBits, int *MaxChannels )`
- Purpose: Detect and configure specific sound card hardware; populate capabilities
- Inputs: Sound card enum / Blaster config structure
- Outputs/Return: Error code; device/capability pointers populated on success
- Side effects: Hardware I/O, probing device capabilities

### FX_PlayVOC / FX_PlayWAV / FX_PlayRaw (and looped variants)
- Signature: `int FX_PlayVOC( char *ptr, int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval )` (non-looped); looped variants add `long loopstart, long loopend` parameters
- Purpose: Queue an audio sample for playback with specified pitch, volume, stereo panning, and priority
- Inputs: Pointer to sample data in memory, pitch offset, volume, left/right pan values, priority for voice allocation, callback value
- Outputs/Return: Voice handle (non-negative) on success, error code on failure
- Side effects: Allocates a voice, schedules playback in mixer; invokes callback when sound ends
- Notes: VOC and WAV are format-aware; Raw requires explicit length and sample rate; higher priority preempts lower-priority voices

### FX_PlayVOC3D / FX_PlayWAV3D
- Signature: `int FX_PlayVOC3D( char *ptr, int pitchoffset, int angle, int distance, int priority, unsigned long callbackval )`
- Purpose: Play a sample with 3D positioning (angle/distance from listener) rather than explicit left/right panning
- Inputs: Sample pointer, pitch offset, angle (horizontal direction), distance, priority, callback
- Outputs/Return: Voice handle or error code
- Side effects: Computes pan/volume from 3D position and applies to voice

### FX_StopSound / FX_StopAllSounds
- Signature: `int FX_StopSound( int handle )` / `int FX_StopAllSounds( void )`
- Purpose: Halt playback of a specific voice or all active voices
- Outputs/Return: Error code
- Side effects: Stops audio, frees voice

### FX_SetPan / FX_SetPitch / FX_SetFrequency / FX_Pan3D
- Signature: `int FX_SetPan( int handle, int vol, int left, int right )` (and pitch/frequency/3D variants)
- Purpose: Modify playback parameters on active voice in real-time
- Inputs: Voice handle, new parameter value(s)
- Outputs/Return: Error code
- Side effects: Updates mixer state for running voice

### FX_SetVolume / FX_GetVolume
- Signature: `void FX_SetVolume( int volume )` / `int FX_GetVolume( void )`
- Purpose: Get/set global master volume
- Side effects: Affects all active voices

### FX_SetReverb / FX_SetFastReverb / FX_SetReverbDelay
- Signature: `void FX_SetReverb( int reverb )` / `void FX_SetFastReverb( int reverb )` / `void FX_SetReverbDelay( int delay )`
- Purpose: Configure spatial reverb effect and delay parameters
- Side effects: Updates mixer reverb buffer and delay line

### FX_StartRecording / FX_StopRecord
- Signature: `int FX_StartRecording( int MixRate, void ( *function )( char *ptr, int length ) )` / `void FX_StopRecord( void )`
- Purpose: Capture mixer output to callback or record ambient audio from input
- Inputs: Mixing rate, callback function receiving buffer pointers and lengths
- Outputs/Return: Error code for start
- Side effects: Allocates recording buffer, invokes callback with recorded chunks

### FX_SetCallBack
- Signature: `int FX_SetCallBack( void ( *function )( unsigned long ) )`
- Purpose: Register a global callback invoked when voices complete
- Inputs: Callback function pointer
- Side effects: Replaces previous callback

### FX_StartDemandFeedPlayback
- Signature: `int FX_StartDemandFeedPlayback( void ( *function )( char **ptr, unsigned long *length ), ... )`
- Purpose: Start playback with dynamic sample data supplied via callback (useful for procedural or streaming audio)
- Inputs: Callback to fetch next buffer, rate, pitch, volume, pan, priority, callback value
- Outputs/Return: Voice handle or error code
- Side effects: Allocates voice, invokes data callback at regular intervals

## Control Flow Notes
**Initialization phase:** `FX_GetBlasterSettings()` → `FX_SetupCard()` → `FX_Init()` to detect and initialize hardware.

**Runtime playback:** `FX_PlayXXX()` queues samples; mixer runs continuously (hardware-interrupt-driven or timer-based). Voice callbacks fire asynchronously when samples end.

**Shutdown:** `FX_Shutdown()` releases resources.

Real-time control: `FX_SetPan()`, `FX_SetPitch()`, `FX_SetVolume()` modify active voices without stopping them.

## External Dependencies
- **Includes:** `sndcards.h` — defines sound card enum (SoundBlaster, Adlib, GenMidi, etc.)
- **Defined elsewhere:** Actual implementation in `FX_MAN.C`; hardware-level sound card I/O drivers; interrupt handlers and DMA management (not visible in header)
