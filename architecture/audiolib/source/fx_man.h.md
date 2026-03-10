# audiolib/source/fx_man.h

## File Purpose
Public header for the sound effects manager (FX_MAN.C) in a 1994 Apogee Software game engine. Provides the primary API for sound card initialization, audio playback, effects processing, spatial audio, and recording functionality.

## Core Responsibilities
- Sound card detection, configuration, and initialization (Sound Blaster support)
- Sound file playback (VOC, WAV, raw formats) with looping and priority management
- Voice management (allocation based on priority, voice availability checks)
- Audio effects (reverb, pitch shifting, frequency control, panning, 3D spatial audio)
- Master volume and audio parameter control
- Sound lifecycle management (stop, callback-based completion notification)
- Demand-feed playback for streaming audio
- Audio recording capture with callback-based sample delivery
- Error handling and device capability reporting

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `fx_device` | struct | Reports device capabilities: max concurrent voices, sample bit depth, channel count (mono/stereo) |
| `fx_blaster_config` | struct | Sound Blaster hardware configuration: I/O address, card type, interrupt, DMA channels, MIDI support, EMU support |
| `FX_ERRORS` | enum | Return codes (FX_Ok, FX_Error, FX_Warning, hardware-specific errors) |
| `fx_BLASTER_Types` | enum | Sound Blaster card variants (SB, SBPro, SB20, SBPro2, SB16) |

## Global / File-Static State
None (header file only; state managed in fx_man.c).

## Key Functions / Methods

### FX_Init
- Signature: `int FX_Init(int SoundCard, int numvoices, int numchannels, int samplebits, unsigned mixrate)`
- Purpose: Initialize the sound effects system with specified sound card and audio parameters.
- Inputs: Sound card ID, number of concurrent voices, channel count (mono/stereo), sample bit depth, mixing rate (Hz).
- Outputs/Return: Error code (FX_Ok on success).
- Side effects: Allocates voice buffers, initializes audio hardware, starts mixer thread or ISR.
- Calls: Not visible; implementation in fx_man.c.
- Notes: Must be called before any audio playback. Determines device capabilities via FX_SetupCard.

### FX_Shutdown
- Signature: `int FX_Shutdown(void)`
- Purpose: Clean shutdown of the sound effects system.
- Inputs: None.
- Outputs/Return: Error code.
- Side effects: Stops all sounds, releases voice buffers, disables hardware interrupts/DMA.
- Calls: Not visible.
- Notes: Inverse of FX_Init; must be called before exit to avoid ISR faults.

### FX_PlayVOC / FX_PlayWAV / FX_PlayRaw
- Signature: `int FX_PlayVOC(char *ptr, int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval)`
- Purpose: Start one-shot playback of a sound file in memory.
- Inputs: Pointer to audio data (VOC/WAV/raw), pitch offset (semitones), volume, stereo pan (left/right), priority (voice stealing), callback user value.
- Outputs/Return: Handle (positive) on success, or error code.
- Side effects: Allocates a voice, starts playback, registers callback for completion notification.
- Calls: Not visible.
- Notes: Handle used to control (stop, pan, pitch) the running sound. Priority determines eviction if no voices available. Callback invoked when sound finishes.

### FX_PlayLoopedVOC / FX_PlayLoopedWAV / FX_PlayLoopedRaw
- Signature: `int FX_PlayLoopedVOC(char *ptr, long loopstart, long loopend, ...)`
- Purpose: Start looping playback of a sound file with loop region definition.
- Inputs: Audio data pointer, loop start/end byte offsets, same playback parameters as non-looped variants.
- Outputs/Return: Voice handle.
- Side effects: Allocates a voice, enables sample looping.
- Notes: Loop points define sample regions; playback repeats from loopstart to loopend indefinitely until FX_StopSound.

### FX_PlayVOC3D / FX_PlayWAV3D
- Signature: `int FX_PlayVOC3D(char *ptr, int pitchoffset, int angle, int distance, int priority, unsigned long callbackval)`
- Purpose: Start 3D spatial audio playback (angle and distance relative to listener).
- Inputs: Audio data, pitch, listener-relative angle (degrees), distance (units), priority, callback value.
- Outputs/Return: Voice handle.
- Side effects: Allocates voice, computes stereo pan and level attenuation from spatial parameters.
- Notes: Angle/distance are converted internally to left/right pan and volume for stereo rendering.

### FX_Pan3D
- Signature: `int FX_Pan3D(int handle, int angle, int distance)`
- Purpose: Update spatial position of a running sound.
- Inputs: Voice handle, new angle and distance.
- Outputs/Return: Status code.
- Side effects: Recalculates pan and attenuation for the voice.
- Notes: Allows real-time movement of sound source (e.g., moving objects in game world).

### FX_SetPan / FX_SetPitch / FX_SetFrequency
- Signature: `int FX_SetPan(int handle, int vol, int left, int right)` / `int FX_SetPitch(int handle, int pitchoffset)` / `int FX_SetFrequency(int handle, int frequency)`
- Purpose: Modify playback parameters of an active voice in real-time.
- Inputs: Voice handle, parameter value (volume/pan levels, pitch offset, frequency in Hz).
- Outputs/Return: Status code.
- Side effects: Updates voice DSP registers or sample pointer.
- Notes: Allows dynamic audio effects during playback (pitch bending, volume automation, panning).

### FX_SetVolume / FX_GetVolume
- Signature: `void FX_SetVolume(int volume)` / `int FX_GetVolume(void)`
- Purpose: Global master volume control.
- Inputs/Outputs: Master volume level (scale unit not specified; likely 0–255).
- Side effects: Scales all active voice output.

### FX_SetReverb / FX_SetReverbDelay / FX_GetReverbDelay / FX_GetMaxReverbDelay
- Signature: `void FX_SetReverb(int reverb)`, `void FX_SetFastReverb(int reverb)`, etc.
- Purpose: Configure reverb effect parameters and delay.
- Inputs: Reverb level/delay in milliseconds.
- Outputs/Return: Current delay or max supported delay.
- Side effects: Modifies audio processing; delay adjustment may affect buffer allocation.

### FX_VoiceAvailable
- Signature: `int FX_VoiceAvailable(int priority)`
- Purpose: Check if a voice slot is available at the given priority level.
- Inputs: Priority level.
- Outputs/Return: Boolean or available voice count.
- Notes: Used before initiating playback to predict success or implement voice stealing policy.

### FX_StopSound / FX_StopAllSounds
- Signature: `int FX_StopSound(int handle)` / `int FX_StopAllSounds(void)`
- Purpose: Stop a specific sound or all active sounds immediately.
- Inputs: Voice handle (or none for all).
- Outputs/Return: Status code.
- Side effects: Releases voice, stops DMA, may or may not fire completion callback.

### FX_StartRecording / FX_StopRecord
- Signature: `int FX_StartRecording(int MixRate, void (*function)(char *ptr, int length))` / `void FX_StopRecord(void)`
- Purpose: Capture audio input to a callback function; stop recording.
- Inputs: Sampling rate, callback function (invoked with audio buffer and sample count).
- Outputs/Return: Status code.
- Side effects: Routes ADC or line-in to callback; recording runs asynchronously.

### FX_StartDemandFeedPlayback
- Signature: `int FX_StartDemandFeedPlayback(void (*function)(char **ptr, unsigned long *length), ...)`
- Purpose: Streaming playback: engine calls callback to request next buffer of audio data.
- Inputs: Callback function, audio parameters (rate, pitch, pan, priority).
- Outputs/Return: Voice handle.
- Side effects: Callback invoked repeatedly to feed samples; allows dynamic or streamed content.
- Notes: Useful for music or procedurally-generated audio.

### FX_SetupCard / FX_GetBlasterSettings / FX_SetupSoundBlaster
- Signature: `int FX_SetupCard(int SoundCard, fx_device *device)` / `int FX_GetBlasterSettings(fx_blaster_config *blaster)` / `int FX_SetupSoundBlaster(fx_blaster_config blaster, ...)`
- Purpose: Detect, query, and configure sound hardware (primarily Sound Blaster).
- Inputs: Sound card type (enum), config struct pointers.
- Outputs/Return: Populated device capabilities or error code.
- Side effects: May probe hardware, read BLASTER environment variable, program I/O ports or DMA.

### FX_ErrorString
- Signature: `char *FX_ErrorString(int ErrorNumber)`
- Purpose: Convert error code to human-readable string for debugging/logging.
- Inputs: Error code from FX_ERRORS enum.
- Outputs/Return: Pointer to static error message string.

## Control Flow Notes
- **Initialization**: Game calls `FX_Init()` at startup; `FX_SetupCard()` and hardware configuration precede this.
- **Gameplay Loop**: Calls to `FX_PlayVOC()`, `FX_Pan3D()`, `FX_SetVolume()` etc. occur on each frame or during event handling (footsteps, weapon fire, ambient audio).
- **Shutdown**: `FX_Shutdown()` called at game exit to clean up hardware and prevent audio glitches.
- **Callbacks**: Completion callbacks (`callbackval`) signal the game engine when a sound finishes, enabling chaining or resource reuse.

## External Dependencies
- **Include**: `sndcards.h` — enumeration of supported sound card types.
- **Defined Elsewhere**: Sound card driver implementations, ISR/DMA handlers, mixer algorithm (all in fx_man.c and subordinate modules).
