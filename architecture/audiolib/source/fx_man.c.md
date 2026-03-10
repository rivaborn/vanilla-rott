# audiolib/source/fx_man.c

## File Purpose
Device-independent sound effects manager providing a high-level API for audio playback, recording, and device abstraction. Acts as a facade layer over multiple sound card drivers (Sound Blaster, PAS16, SoundScape, Ultrasound, SoundSource), delegating most functionality to the MULTIVOC mixer library.

## Core Responsibilities
- Sound device initialization, configuration, and shutdown across multiple hardware types
- Sound playback in multiple formats (VOC, WAV, raw) with looping and 3D positioning
- Voice management (voice availability checking, playback control)
- Audio property manipulation (volume, panning, pitch, frequency, reverb)
- Recording initiation and termination
- Error code mapping to user-readable messages
- Device-specific error handling and delegation to appropriate driver

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `fx_device` | struct | Device capability info (MaxVoices, MaxSampleBits, MaxChannels) – defined elsewhere |
| `fx_blaster_config` | struct | Sound Blaster configuration wrapper (type, address, interrupt, DMA channels, MIDI, emulation flags) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FX_MixRate` | unsigned | static | Current mixing sample rate in Hz |
| `FX_SoundDevice` | int | global | Currently active sound card (enum from sndcards.h; -1 if none) |
| `FX_ErrorCode` | int | global | Last error code for retrieval via FX_ErrorString |
| `FX_Installed` | int | global | Boolean flag indicating whether audio system is initialized |

## Key Functions / Methods

### FX_ErrorString
- **Signature:** `char *FX_ErrorString(int ErrorNumber)`
- **Purpose:** Map error codes to human-readable error messages; handles recursive lookup for FX_Warning/FX_Error codes
- **Inputs:** ErrorNumber – error code (may be -1 for current error via FX_ErrorCode)
- **Outputs/Return:** Pointer to static error message string
- **Side effects:** None (pure lookup function)
- **Calls:** BLASTER_ErrorString, PAS_ErrorString, SOUNDSCAPE_ErrorString, GUSWAVE_ErrorString, SS_ErrorString, MV_ErrorString
- **Notes:** Switch statement delegates to device-specific error mappers based on FX_SoundDevice; recursive for FX_Warning/Error codes

### FX_SetupCard
- **Signature:** `int FX_SetupCard(int SoundCard, fx_device *device)`
- **Purpose:** Initialize and query a specific sound card device without full system init; populates device capability info
- **Inputs:** SoundCard (enum), device (output struct)
- **Outputs/Return:** FX_Ok on success, FX_Error on init failure
- **Side effects:** Sets FX_SoundDevice, modifies device struct, may lock DMA/IRQ resources via device drivers
- **Calls:** BLASTER_Init, BLASTER_GetCardInfo, PAS_Init, PAS_GetCardInfo, SOUNDSCAPE_GetCardInfo, GUSWAVE_Init, SS_Init, SS_Shutdown
- **Notes:** GenMidi/SoundCanvas/WaveBlaster set zero capabilities; MIDI-only devices cannot play samples; checks USER_CheckParameter("ASSVER") to return version info

### FX_Init
- **Signature:** `int FX_Init(int SoundCard, int numvoices, int numchannels, int samplebits, unsigned mixrate)`
- **Purpose:** Initialize the sound system with chosen device and mixer parameters; locks memory and delegates to MV_Init
- **Inputs:** SoundCard, numvoices, numchannels, samplebits, mixrate (sample rate in Hz)
- **Outputs/Return:** FX_Ok on success, FX_Error if memory lock or MV_Init fails
- **Side effects:** Calls FX_Shutdown if already installed; locks low-level memory via LL_LockMemory; sets FX_MixRate; sets FX_Installed = TRUE on success
- **Calls:** FX_Shutdown, LL_LockMemory, MV_Init, LL_UnlockMemory
- **Notes:** Only supports cards handled by MULTIVOC library (SoundBlaster, Awe32, PAS, SoundScape, SoundSource, TandySoundSource, UltraSound); returns FX_InvalidCard for others

### FX_Shutdown
- **Signature:** `int FX_Shutdown(void)`
- **Purpose:** Terminate audio playback, halt mixer, unlock memory, and reset installed flag
- **Inputs:** None
- **Outputs/Return:** FX_Ok on success, FX_Error if MV_Shutdown fails
- **Side effects:** Calls MV_Shutdown, LL_UnlockMemory; sets FX_Installed = FALSE
- **Calls:** MV_Shutdown, LL_UnlockMemory
- **Notes:** No-op if FX_Installed is already FALSE; must match FX_Init calls

### FX_SetCallback
- **Signature:** `int FX_SetCallBack(void (*function)(unsigned long))`
- **Purpose:** Register a callback function to invoke when a voice finishes playback
- **Inputs:** function pointer (takes callback value as unsigned long)
- **Outputs/Return:** FX_Ok on success, FX_Error if invalid device
- **Side effects:** Delegates to MV_SetCallBack to register callback in mixer
- **Calls:** MV_SetCallBack
- **Notes:** Callback receives the callbackval passed to playback functions; only supported for MV-based devices

### FX_PlayVOC / FX_PlayLoopedVOC / FX_PlayWAV / FX_PlayLoopedWAV / FX_PlayRaw / FX_PlayLoopedRaw / FX_PlayVOC3D / FX_PlayWAV3D
- **Signature:** Various (see file; examples: `int FX_PlayVOC(char *ptr, int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval)`)
- **Purpose:** Initiate playback of audio data in various formats; return voice handle for later control
- **Inputs:** Audio buffer pointer, playback parameters (pitch offset, volume, stereo panning, priority), optional loop boundaries for looped variants, angle/distance for 3D variants
- **Outputs/Return:** Voice handle (positive integer) on success, FX_Warning if MV_* returns error
- **Side effects:** Allocates voice resource in mixer; registers callback; starts audio playback
- **Calls:** Corresponding MV_Play* function
- **Notes:** 3D variants (VOC3D, WAV3D) use angle/distance instead of left/right channels; looped variants specify loop boundaries; priority determines which sounds are culled if voice limit exceeded

### FX_StopSound / FX_StopAllSounds
- **Signature:** `int FX_StopSound(int handle)`, `int FX_StopAllSounds(void)`
- **Purpose:** Halt playback of a specific voice or all active voices
- **Inputs:** Voice handle (for FX_StopSound)
- **Outputs/Return:** FX_Ok on success, FX_Warning if MV_Kill/MV_KillAllVoices fails
- **Side effects:** Terminates audio playback; frees voice resource
- **Calls:** MV_Kill, MV_KillAllVoices
- **Notes:** No error is set; only warning returned; safe to call on invalid handles

### FX_SetVolume / FX_GetVolume
- **Signature:** `void FX_SetVolume(int volume)`, `int FX_GetVolume(void)`
- **Purpose:** Control global audio output level; may use hardware mixer or software mixer depending on device
- **Inputs:** volume (0-255 typically)
- **Outputs/Return:** void (set), int volume level (get)
- **Side effects:** Modifies device mixer or MV software volume
- **Calls:** BLASTER_CardHasMixer, BLASTER_SetVoiceVolume, MV_SetVolume, PAS_SetPCMVolume, PAS_GetPCMVolume, GUSWAVE_SetVolume, GUSWAVE_GetVolume
- **Notes:** Sound Blaster uses hardware mixer if available, else falls back to MV; PAS16 falls back to MV if hardware fails; MIDI devices return fixed volume (255); GUS uses GUSWAVE driver

### FX_SetPan / FX_SetPitch / FX_SetFrequency
- **Signature:** `int FX_SetPan(int handle, int vol, int left, int right)`, `int FX_SetPitch(int handle, int pitchoffset)`, `int FX_SetFrequency(int handle, int frequency)`
- **Purpose:** Modify playback properties of an active voice
- **Inputs:** Voice handle, new value (pan: separate left/right levels; pitch: offset; frequency: Hz)
- **Outputs/Return:** FX_Ok on success, FX_Warning if MV_* fails
- **Side effects:** Updates mixer state for specified voice
- **Calls:** MV_SetPan, MV_SetPitch, MV_SetFrequency
- **Notes:** All delegate directly to MULTIVOC layer

### FX_Pan3D
- **Signature:** `int FX_Pan3D(int handle, int angle, int distance)`
- **Purpose:** Update 3D position (angle and distance from listener) for active voice
- **Inputs:** Voice handle, angle (0-360 degrees), distance
- **Outputs/Return:** FX_Ok on success, FX_Warning if MV_Pan3D fails
- **Side effects:** Recalculates stereo panning based on 3D position
- **Calls:** MV_Pan3D
- **Notes:** Used by games to implement spatial audio as listener/sound source moves

### FX_StartRecording / FX_StopRecord
- **Signature:** `int FX_StartRecording(int MixRate, void (*function)(char *ptr, int length))`, `void FX_StopRecord(void)`
- **Purpose:** Initiate audio input from sound card to application buffer; callback-driven buffering
- **Inputs:** Desired mix rate, callback function
- **Outputs/Return:** FX_Ok/FX_Warning (start), void (stop)
- **Side effects:** Sets up ADC hardware; registers callback for audio chunk delivery
- **Calls:** MV_StartRecording, MV_StopRecord
- **Notes:** Recording only supported on Blaster, Awe32, PAS, SoundMan16; stops on unsupported devices

### FX_GetBlasterSettings
- **Signature:** `int FX_GetBlasterSettings(fx_blaster_config *blaster)`
- **Purpose:** Query BLASTER environment variable settings for manual configuration
- **Inputs:** Output struct
- **Outputs/Return:** FX_Ok on success, FX_Error if BLASTER_GetEnv fails
- **Side effects:** Populates blaster struct
- **Calls:** BLASTER_GetEnv
- **Notes:** Used to read DOS BLASTER env var for hardware detection

### FX_SetupSoundBlaster
- **Signature:** `int FX_SetupSoundBlaster(fx_blaster_config blaster, int *MaxVoices, int *MaxSampleBits, int *MaxChannels)`
- **Purpose:** Manually configure Sound Blaster with explicit settings instead of auto-detection
- **Inputs:** Blaster config, output capability pointers
- **Outputs/Return:** FX_Ok on success, FX_Error if init fails
- **Side effects:** Sets FX_SoundDevice = SoundBlaster; calls BLASTER_SetCardSettings, BLASTER_Init; populates capability outputs
- **Calls:** BLASTER_SetCardSettings, BLASTER_Init, BLASTER_GetCardInfo
- **Notes:** Used when BLASTER env var is unavailable or incorrect; sets MaxVoices = 8 (not 32 like FX_SetupCard)

## Control Flow Notes
This is a **device initialization and command API**, not frame-driven. Flow:
1. **Init phase:** User calls `FX_Init(device, params)` → locks memory, delegates to `MV_Init`
2. **Playback phase:** User calls `FX_PlayVOC`/`FX_PlayWAV`/etc. → returns voice handle; mixer streams audio in background
3. **Control phase:** User adjusts voice properties (`FX_SetVolume`, `FX_SetPan`, `FX_SetPitch`, `FX_Pan3D`)
4. **Cleanup phase:** User calls `FX_StopSound` (per-voice) or `FX_StopAllSounds`, then `FX_Shutdown` → unlocks memory

The device dispatcher pattern (switch on `FX_SoundDevice`) ensures device-specific code paths. Most playback logic is delegated to MULTIVOC (`MV_*` functions); device drivers handle only initialization, volume control, and raw mixer control.

## External Dependencies
- **Standard Library:** stdio.h, stdlib.h
- **Sound card drivers:** blaster.h, pas16.h, sndscape.h, guswave.h, sndsrc.h
- **Core mixer:** multivoc.h (MULTIVOC library – does actual voice mixing and playback)
- **Device enumeration:** sndcards.h (sound card type constants)
- **Memory management:** ll_man.h (low-level DMA/IRQ memory locking)
- **User input:** user.h (USER_CheckParameter for command-line/env var checks)
- **Self:** fx_man.h (this module's public interface)

**Defined elsewhere:** All device-specific drivers (BLASTER, PAS, SOUNDSCAPE, GUSWAVE, SS), MULTIVOC mixer, memory locker (LL_*), user parameter parser
