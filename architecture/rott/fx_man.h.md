# rott/fx_man.h

## File Purpose

Public header for the effects/sound manager subsystem. Declares the interface for hardware-accelerated sound playback, voice management, and real-time audio control (volume, panning, pitch, 3D positioning, reverb). Supports Sound Blaster and compatible sound cards with both 2D and 3D audio mixing.

## Core Responsibilities

- Sound card hardware initialization and detection (particularly Sound Blaster variants)
- Multi-voice sound mixing and playback with priority-based voice allocation
- Audio effect playback for VOC, WAV, and raw PCM formats with loop support
- Real-time sound parameter control (volume, pan, pitch, frequency, reverb)
- 3D audio positioning (angle/distance-based panning and attenuation)
- Callback-driven completion notification for sound events
- Live audio recording at configurable sample rates
- Demand-feed playback for streaming/procedural audio

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `fx_device` | struct | Describes sound device capabilities (max voices, sample bits, channels) |
| `fx_blaster_config` | struct | Sound Blaster hardware configuration (I/O address, IRQ, DMA channels, MIDI, EMU8000) |
| `FX_ERRORS` | enum | Status/error codes for function returns (Warning, Error, Ok, version/card/DPMI errors) |
| `fx_BLASTER_Types` | enum | Sound Blaster card variants (SB, SBPro, SB20, SBPro2, SB16) |

## Global / File-Static State

None.

## Key Functions / Methods

### FX_Init
- Signature: `int FX_Init( int SoundCard, int numvoices, int numchannels, int samplebits, unsigned mixrate )`
- Purpose: Initialize the effects engine with specified hardware and mixing parameters
- Inputs: Sound card type, voice count, channel count (mono/stereo), sample bit depth, mix rate in Hz
- Outputs/Return: FX_ERRORS status code
- Side effects: Initializes sound hardware, allocates voice buffers, prepares mixer
- Notes: Must be called before any playback functions

### FX_Shutdown
- Signature: `int FX_Shutdown( void )`
- Purpose: Shut down the effects engine and release hardware resources
- Outputs/Return: FX_ERRORS status code
- Side effects: Stops all playback, deallocates buffers, resets sound hardware

### FX_PlayVOC, FX_PlayWAV, FX_PlayRaw
- Signature: `int FX_Play{VOC|WAV|Raw}( char *ptr, [dimensions], int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval )`
- Purpose: Queue a sound sample for immediate playback
- Inputs: Sound data pointer, format-specific fields (pitch offset, volume, stereo pan, priority level, callback value)
- Outputs/Return: Voice handle (>= 0) or FX_ERRORS code
- Side effects: Allocates voice if available, begins DMA/mixer operation
- Notes: Priority determines voice preemption when all voices occupied; callback fires on completion

### FX_PlayVOC3D, FX_PlayWAV3D
- Signature: `int FX_Play{VOC|WAV}3D( char *ptr, int pitchoffset, int angle, int distance, int priority, unsigned long callbackval )`
- Purpose: Play a sound with 3D positioning (listener-relative angle and distance)
- Inputs: Sample pointer, pitch, angle (0–359°), distance, priority, callback value
- Outputs/Return: Voice handle or error
- Notes: Engine internally converts angle/distance to stereo pan and attenuation

### FX_PlayLoopedVOC, FX_PlayLoopedWAV, FX_PlayLoopedRaw
- Signature: `int FX_PlayLooped{VOC|WAV|Raw}( char *ptr, long loopstart, long loopend, ... )`
- Purpose: Play a sample repeatedly between loop markers
- Inputs: Sample pointer, loop region bounds (byte offsets), standard playback parameters
- Outputs/Return: Voice handle or error
- Side effects: Sets up hardware or software looping
- Calls: FX_EndLooping to terminate

### FX_SetPan, FX_SetPitch, FX_SetFrequency
- Purpose: Modify active playback parameters
- Inputs: Voice handle, new parameter value(s)
- Outputs/Return: FX_ERRORS status
- Notes: FX_SetPan takes explicit left/right volumes; FX_Pan3D re-converts angle/distance

### FX_Pan3D
- Signature: `int FX_Pan3D( int handle, int angle, int distance )`
- Purpose: Update 3D positioning of an active sound
- Notes: Used for moving sound sources in real-time (e.g., moving enemies)

### FX_SetVolume, FX_SetReverb, FX_SetReverseStereo
- Purpose: Global mixer settings (master volume, reverb depth, stereo polarity)
- Notes: FX_SetFastReverb and reverb delay functions fine-tune reverb characteristics

### FX_VoiceAvailable
- Signature: `int FX_VoiceAvailable( int priority )`
- Purpose: Check if a voice at given priority level can be allocated
- Outputs/Return: Voice handle if available, or error code

### FX_SoundActive, FX_SoundsPlaying
- Purpose: Query playback status (single voice or total count)
- Notes: Used for cleanup and state checking

### FX_StopSound, FX_StopAllSounds
- Purpose: Halt playback on a specific voice or all voices
- Outputs/Return: FX_ERRORS status

### FX_StartDemandFeedPlayback
- Signature: `int FX_StartDemandFeedPlayback( void ( *function )( char **ptr, unsigned long *length ), ... )`
- Purpose: Begin streaming playback with callback-driven data feeding
- Inputs: Callback function that supplies buffers on demand, standard playback parameters
- Outputs/Return: Voice handle or error
- Side effects: Invoke callback repeatedly during playback to refill buffers

### FX_StartRecording, FX_StopRecord
- Purpose: Record mixed audio output to memory
- Inputs: Sample rate and completion callback
- Side effects: Diverts mixer output to recording buffer

## Control Flow Notes

**Initialization phase:** `FX_GetBlasterSettings()` → `FX_SetupSoundBlaster()` or `FX_SetupCard()` → `FX_Init()` during engine startup.

**Gameplay frame:** Game code calls `FX_Play*()` functions to queue sounds; `FX_Pan3D()` updates 3D-positioned sounds each frame; voice callbacks fire asynchronously.

**Shutdown phase:** `FX_StopAllSounds()` → `FX_Shutdown()` on engine exit.

## External Dependencies

- **sndcards.h**: Sound card enumeration (`soundcardnames` enum for card type constants)
- Implied: Low-level hardware drivers, mixer libraries (not declared here)
