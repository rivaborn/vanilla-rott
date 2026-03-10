# audiolib/source/multivoc.c

## File Purpose

Core multi-voice audio mixing engine for DOS-era Sound Blaster and compatible sound cards. Manages voice allocation, mixing, playback across multiple audio formats (VOC, WAV, RAW), and sound effects like reverb and 3D panning using interrupt-driven DMA-based circular buffers.

## Core Responsibilities

- Initialize and manage multi-voice playback system with hardware sound card drivers
- Allocate and manage a pool of voice nodes with priority-based preemption
- Mix multiple active voices into circular DMA buffers at regular intervals
- Parse and playback audio in VOC, WAV, and raw formats with looping support
- Support demand-fed (callback-based) audio for streaming playback
- Apply per-voice volume, pitch, stereo pan, and reverb effects via lookup tables
- Handle sound card hardware abstractions (Sound Blaster, Gravis UltraSound, Pro Audio Spectrum, etc.)
- Manage error codes and user callbacks when voices finish playback
- Implement recording mode for supported sound cards
- Lock/unlock critical real-time code and data to physical memory (DPMI)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| VoiceNode | struct | (defined in _multivc.h) Voice state: playback position, rate, volume, format, loop info, GetSound callback |
| Pan | struct | (defined elsewhere) Stereo pan levels: left and right channel values |
| VOLUME16 | typedef | (defined elsewhere) 16-bit signed sample for reverb lookup table |
| riff_header | struct | WAV file RIFF container header (RIFF magic, WAVE format, fmt chunk) |
| format_header | struct | WAV fmt chunk: format tag, channels, sample rate, bits per sample |
| data_header | struct | WAV data chunk: sample data size and pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MV_ReverbLevel | int | static | Reverb effect intensity (0 = disabled) |
| MV_ReverbDelay | int | static | Reverb delay in bytes (circular buffer wrap offset) |
| MV_ReverbTable | VOLUME16 * | static | Pointer to reverb attenuation lookup table |
| MV_VolumeTable | signed short[64][256] | static | Volume scaling tables: 64 levels × 256 input values |
| MV_PanTable | Pan[32][64] | static | Stereo pan tables: 32 angles × 64 distance/volume levels |
| MV_Installed | int | static | System initialization flag |
| MV_SoundCard | int | static | Enum value of active sound card type |
| MV_TotalVolume | int | static | Master volume level (0–255) |
| MV_MaxVoices | int | static | Max concurrent voices allocated |
| MV_Recording | int | static | Recording mode active flag |
| MV_BufferSize | int | static | Single mix buffer size in bytes |
| MV_BufferLength | int | static | Total circular buffer size |
| MV_NumberOfBuffers | int | static | Number of circular buffers (typically 2–4) |
| MV_MixMode | int | static | Bitmask: MONO_8BIT, MONO_16BIT, STEREO_8BIT, STEREO_16BIT |
| MV_Channels | int | static | 1 (mono) or 2 (stereo) |
| MV_Bits | int | static | 8 or 16 bits per sample |
| MV_Silence | int | static | Silence fill byte/word for current format |
| MV_SwapLeftRight | int | static | Reverse stereo (Sound Blaster Pro compatibility) |
| MV_RequestedMixRate | int | static | Requested playback sample rate (Hz) |
| MV_MixRate | int | static | Actual achieved sample rate |
| MV_DMAChannel | int | static | DMA channel used (−1 if not applicable) |
| MV_BuffShift | int | static | Bit shift for buffer address calculations |
| MV_TotalMemory | int | static | Total allocated voice/clip memory |
| MV_BufferDescriptor | int | static | DPMI DOS memory descriptor handle |
| MV_BufferEmpty | int[NumberOfBuffers] | static | Silence flags per buffer |
| MV_MixBuffer | char *[NumberOfBuffers+1] | global | Mix buffer pointers (circular) |
| MV_Voices | VoiceNode * | static | Voice pool array base pointer |
| VoiceList | VoiceNode volatile | static | List head for active voices (linked, sorted by priority) |
| VoicePool | VoiceNode volatile | static | List head for free voice nodes |
| MV_MixPage | int | static | Current mix buffer index (0 to NumberOfBuffers−1) |
| MV_VoiceHandle | int | static | Incrementing handle counter for voice allocation |
| MV_CallBackFunc | void(*)(unsigned long) | static | User callback invoked when voice ends |
| MV_RecordFunc | void(*)(char *, int) | static | User record callback |
| MV_MixFunction | void(*)(VoiceNode *, int) | static | Pointer to active mix function (locked code) |
| MV_MaxVolume | int | static | Max volume level (63) |
| MV_HarshClipTable | char * | global | Audio clipping table (256 + 128 + 256 entry ranges) |
| MV_MixDestination | char * | global | Current output write pointer during mix |
| MV_LeftVolume | short * | global | Left volume table pointer during mix |
| MV_RightVolume | short * | global | Right volume table pointer during mix |
| MV_SampleSize | int | global | Bytes per sample (1–4 depending on channels/bits) |
| MV_RightChannelOffset | int | global | Offset to right channel in buffer |
| MV_MixPosition | unsigned long | global | Fixed-point position during mixing |
| MV_ErrorCode | int | global | Current error code (for MV_ErrorString) |

## Key Functions / Methods

### MV_Init
- Signature: `int MV_Init(int soundcard, int MixRate, int Voices, int numchannels, int samplebits)`
- Purpose: Initialize the entire audio subsystem, allocate resources, detect and initialize sound card hardware
- Inputs: sound card type enum, desired mix rate (Hz), max voices, channel count (1–2), sample bits (8–16)
- Outputs/Return: MV_Ok on success; MV_Error with error code set otherwise
- Side effects: Allocates voice pool and DMA buffers, locks critical code/data to physical memory, calls sound card init routines, starts playback engine
- Calls: MV_LockMemory, USRHOOKS_GetMem, DPMI_LockMemory, DPMI_GetDOSMemory, card-specific init (BLASTER_Init, GUSWAVE_Init, etc.), MV_SetMixMode, MV_CalcPanTable, MV_CalcVolume, MV_StartPlayback, MV_TestPlayback
- Notes: Fails if no free memory, if sound card cannot be initialized, or if DMA/IRQ test fails; calls MV_Shutdown on failure

### MV_Shutdown
- Signature: `int MV_Shutdown(void)`
- Purpose: Shut down the audio system and release all resources
- Inputs: None
- Outputs/Return: MV_Ok
- Side effects: Kills all voices, stops playback engine, shuts down sound card, frees voice and buffer memory, unlocks locked code/data
- Calls: MV_KillAllVoices, MV_StopPlayback, card-specific shutdown, DPMI_UnlockMemory, USRHOOKS_FreeMem, MV_UnlockMemory
- Notes: Safe to call even if not initialized; clears global pointers to prevent use-after-free

### MV_StartPlayback
- Signature: `int MV_StartPlayback(void)`
- Purpose: Start the DMA playback engine for the selected sound card
- Inputs: None (uses global configuration)
- Outputs/Return: MV_Ok on success; MV_Error if card setup fails
- Side effects: Initializes mix buffers to silence, sets MV_MixFunction to MV_Mix, calls card-specific buffered playback setup
- Calls: ClearBuffer_DW, BLASTER_BeginBufferedPlayback, GUSWAVE_StartDemandFeedPlayback, PAS_BeginBufferedPlayback, SOUNDSCAPE_BeginBufferedPlayback, SS_BeginBufferedPlayback
- Notes: Different cards use different playback models (DMA vs. demand-feed); GUS may use two channels for stereo

### MV_ServiceVoc
- Signature: `void MV_ServiceVoc(void)`
- Purpose: Service a DMA buffer boundary interrupt; mix next buffer and handle voice state
- Inputs: None (called from hardware interrupt context)
- Outputs/Return: None
- Side effects: Reads current DMA position, increments MV_MixPage, clears or applies reverb to mix buffer, calls MV_MixFunction for each active voice, stops finished voices, fires user callbacks
- Calls: DMA_GetCurrentPos, ClearBuffer_DW, MV_16BitReverb, MV_16BitReverbFast, MV_8BitReverb, MV_8BitReverbFast, MV_MixFunction (voice→mix), MV_StopVoice, user callback
- Notes: **Memory-locked for real-time safety**; reverb is applied by mixing the delayed buffer; voices without GetSound data are stopped

### MV_Mix
- Signature: `static void MV_Mix(VoiceNode *voice, int buffer)`
- Purpose: Core mixer for a single voice into a single buffer (called from MV_ServiceVoc)
- Inputs: Voice node with position, rate, sound data pointer; buffer index
- Outputs/Return: None
- Side effects: Increments voice→position, calls voice→GetSound if needed, updates MV_MixPosition
- Calls: voice→GetSound, voice→mix (function pointer to bit/channel-specific mixer)
- Notes: **Memory-locked**; handles fixed-point rate scaling; loops over voice blocks until buffer is filled; stops if GetSound returns NoMoreData

### MV_PlayVoice / MV_StopVoice
- Signatures: `void MV_PlayVoice(VoiceNode *voice)` / `void MV_StopVoice(VoiceNode *voice)`
- Purpose: Add voice to play list (sorted by priority) / remove voice from play list and return to free pool
- Inputs: Pointer to voice node
- Outputs/Return: None
- Side effects: Modifies VoiceList/VoicePool linked lists; disables interrupts for safety
- Calls: DisableInterrupts, LL_SortedInsertion, LL_Remove, LL_Add, RestoreInterrupts
- Notes: Interrupt-safe; priority determines preemption order

### MV_AllocVoice
- Signature: `VoiceNode *MV_AllocVoice(int priority)`
- Purpose: Allocate a voice from the free pool, or preempt a lower-priority active voice
- Inputs: Priority level for new voice
- Outputs/Return: Pointer to allocated voice node; NULL if no voice available and cannot preempt
- Side effects: Removes voice from VoicePool or kills lower-priority voice; assigns unique handle; disables/restores interrupts
- Calls: MV_Recording check, LL_Empty, MV_Kill, MV_VoicePlaying
- Notes: Returns NULL if recording is active; increments MV_VoiceHandle until unused handle found

### MV_PlayRaw / MV_PlayLoopedRaw
- Signatures: `int MV_PlayRaw(char *ptr, unsigned long length, unsigned rate, int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval)` / looped variant
- Purpose: Start playback of raw PCM audio with optional looping
- Inputs: Data pointer, length, sample rate, pitch offset, volume, pan, priority, callback value
- Outputs/Return: Voice handle on success; MV_Error if no voices available
- Side effects: Allocates voice, initializes GetSound to MV_GetNextRawBlock, adds to play list
- Calls: MV_AllocVoice, MV_SetVoicePitch, MV_SetVoiceVolume, MV_PlayVoice
- Notes: Calls MV_PlayLoopedRaw internally with NULL loop pointers for non-looped variant

### MV_PlayWAV / MV_PlayLoopedWAV
- Signatures: `int MV_PlayWAV(char *ptr, int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval)` / looped variant
- Purpose: Parse and play a WAV file (PCM, mono, 8 or 16-bit) with optional loop points
- Inputs: WAV file pointer, pitch, volume, pan, priority, callback; looped variant adds absolute loop start/end samples
- Outputs/Return: Voice handle; MV_Error if format invalid or no voices available
- Side effects: Validates WAV headers (RIFF magic, fmt chunk, PCM format, mono, 8/16-bit), allocates voice, initializes GetSound to MV_GetNextWAVBlock
- Calls: strncmp (header validation), MV_AllocVoice, MV_SetVoicePitch, MV_SetVoiceVolume, MV_PlayVoice
- Notes: Converts loopstart/loopend byte offsets accounting for 16-bit samples; handles wrap-around in circular buffer

### MV_PlayVOC / MV_PlayLoopedVOC
- Signatures: `int MV_PlayVOC(char *ptr, int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval)` / looped variant
- Purpose: Parse and play a Creative Labs VOC file with optional loop points
- Inputs: VOC file pointer, pitch, volume, pan, priority, callback; looped variant adds relative loop start/end byte offsets
- Outputs/Return: Voice handle; MV_Error if not valid VOC or no voices
- Side effects: Validates "Creative Voice File" header, allocates voice, initializes GetSound to MV_GetNextVOCBlock
- Calls: strncmp (header check), MV_AllocVoice, MV_SetVoiceVolume, MV_PlayVoice
- Notes: VOC format supports multiple audio blocks; loopstart/loopend stored as relative pointers converted to absolute in MV_GetNextVOCBlock

### MV_PlayWAV3D / MV_PlayVOC3D
- Signatures: `int MV_PlayWAV3D(char *ptr, int pitchoffset, int angle, int distance, int priority, unsigned long callbackval)` / VOC variant
- Purpose: Play WAV/VOC with 3D spatial audio based on angle and distance from listener
- Inputs: Audio file pointer, pitch, angle (0–31), distance (0–255), priority, callback
- Outputs/Return: Voice handle; MV_Error if not installed or no voices
- Side effects: Calculates pan and volume from 3D pan table based on angle and distance; calls MV_PlayWAV/MV_PlayVOC with calculated values
- Calls: MIX_VOLUME macro, MV_PanTable lookup, MV_PlayWAV, MV_PlayVOC
- Notes: Negative distance reverses angle by 180°; pan table provides smooth left/right and front/back localization

### MV_StartDemandFeedPlayback
- Signature: `int MV_StartDemandFeedPlayback(void (*function)(char **ptr, unsigned long *length), int rate, int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval)`
- Purpose: Start playback from a user-supplied callback function that provides audio data on demand
- Inputs: Callback function pointer, sample rate, pitch, volume, pan, priority, callback value
- Outputs/Return: Voice handle; MV_Error if no voices
- Side effects: Allocates voice, initializes GetSound to MV_GetNextDemandFeedBlock, calls callback to fetch first block
- Calls: MV_AllocVoice, MV_SetVoicePitch, MV_SetVoiceVolume, MV_PlayVoice
- Notes: Callback is invoked from MV_Mix during playback with pointers to fill; supports streaming scenarios

### MV_GetNextVOCBlock / MV_GetNextWAVBlock / MV_GetNextRawBlock / MV_GetNextDemandFeedBlock
- Signatures: Various GetSound callbacks
- Purpose: Fetch the next block of audio data for a voice based on format (VOC, WAV, raw, or demand-fed)
- Outputs/Return: KeepPlaying if more data available; NoMoreData if end reached
- Side effects: Parses block headers, updates voice→sound pointer and length, handles looping, calls user demand-feed callback
- Notes: 
  - **MV_GetNextVOCBlock**: Parses VOC block types (0–9), handles loop markers, skips unsupported formats (stereo, packed, silence)
  - **MV_GetNextWAVBlock**: Simple iteration through WAV data, applies loop start/end
  - **MV_GetNextRawBlock**: Linear playback with optional loop restart
  - **MV_GetNextDemandFeedBlock**: Calls user callback via voice→DemandFeed

### MV_Kill / MV_KillAllVoices
- Signatures: `int MV_Kill(int handle)` / `int MV_KillAllVoices(void)`
- Purpose: Stop a specific voice by handle / stop all active voices
- Inputs: Voice handle (or none for kill-all)
- Outputs/Return: MV_Ok or MV_Error; fires user callback
- Side effects: Removes voice from play list, returns to free pool, calls user callback with voice's callbackval
- Calls: MV_GetVoice, MV_StopVoice, user callback
- Notes: Interrupt-safe; callback fired outside interrupt lock

### MV_SetVoiceVolume
- Signature: `void MV_SetVoiceVolume(VoiceNode *voice, int vol, int left, int right)`
- Purpose: Set the stereo volume levels for a voice
- Inputs: Voice pointer, mono volume, left pan level, right pan level
- Outputs/Return: None
- Side effects: Looks up volume tables, assigns to voice→LeftVolume and voice→RightVolume pointers, recalculates mix mode
- Calls: MV_GetVolumeTable, MV_SetVoiceMixMode
- Notes: If mono output, left=right=vol; SB Pro compatibility mode swaps channels

### MV_SetVoicePitch
- Signature: `void MV_SetVoicePitch(VoiceNode *voice, unsigned long rate, int pitchoffset)`
- Purpose: Set the playback sample rate and pitch offset for a voice
- Inputs: Voice pointer, sample rate (Hz), pitch offset
- Outputs/Return: None
- Side effects: Updates voice→SamplingRate, RateScale, FixedPointBufferSize
- Calls: PITCH_GetScale
- Notes: RateScale = (rate × pitch_scale) / MV_MixRate (fixed-point resampling)

### MV_SetPan / MV_Pan3D
- Signatures: `int MV_SetPan(int handle, int vol, int left, int right)` / `int MV_Pan3D(int handle, int angle, int distance)`
- Purpose: Set stereo pan / 3D pan from angle and distance
- Inputs: Voice handle, volume/pan values; 3D variant uses angle (0–31) and distance (0–255)
- Outputs/Return: MV_Ok on success; MV_Warning or MV_Error if voice not found
- Side effects: Calls MV_SetVoiceVolume with calculated pan values
- Calls: MV_GetVoice, MV_SetVoiceVolume, MV_PanTable lookup
- Notes: 3D pan looks up left/right from MV_PanTable[angle][distance]

### MV_SetReverb / MV_SetFastReverb / MV_SetReverbDelay
- Signatures: `void MV_SetReverb(int reverb)` / `void MV_SetFastReverb(int reverb)` / `void MV_SetReverbDelay(int delay)`
- Purpose: Enable reverb effect / enable fast (approximate) reverb / set reverb delay time
- Inputs: Reverb level (0–255) or (0–16 for fast), delay in samples
- Outputs/Return: None
- Side effects: Updates MV_ReverbLevel, MV_ReverbTable pointer, MV_ReverbDelay
- Calls: MIX_VOLUME macro
- Notes: Standard reverb uses lookup table (slower); fast reverb uses approximation; delay wraps in circular buffer

### MV_SetVolume / MV_GetVolume
- Signatures: `void MV_SetVolume(int volume)` / `int MV_GetVolume(void)`
- Purpose: Set/get master output volume
- Inputs: Master volume level (0–255)
- Outputs/Return: None / master volume
- Side effects: Recalculates all MV_VolumeTable entries via MV_CalcVolume
- Calls: MV_CalcVolume
- Notes: Bounded to [0, MV_MaxTotalVolume]

### MV_SetCallBack
- Signature: `void MV_SetCallBack(void (*function)(unsigned long))`
- Purpose: Register a callback function invoked when any voice finishes
- Inputs: Function pointer
- Outputs/Return: None
- Side effects: Sets global MV_CallBackFunc
- Notes: Callback receives voice's callbackval

### MV_SetMixMode
- Signature: `int MV_SetMixMode(int numchannels, int samplebits)`
- Purpose: Configure output format (mono/stereo, 8/16-bit)
- Inputs: Channel count (1 or 2), sample bits (8 or 16)
- Outputs/Return: MV_Ok or MV_Error
- Side effects: Calls card-specific SetMixMode, recalculates MV_Channels, MV_Bits, MV_BufferSize, MV_SampleSize, MV_RightChannelOffset
- Calls: Card-specific SetMixMode (BLASTER_SetMixMode, etc.)
- Notes: Updates buffer layout for UltraSound stereo (separate buffers for L/R)

### MV_ErrorString
- Signature: `char *MV_ErrorString(int ErrorNumber)`
- Purpose: Return human-readable error message for an error code
- Inputs: Error code or MV_Error / MV_Warning for current error
- Outputs/Return: Pointer to error string
- Side effects: None
- Notes: Switch on 40+ error codes; delegates to card-specific error strings for card errors

### Utility Functions (Internal)
- **MV_CreateVolumeTable**: Pre-calculate volume scaling table for a given level
- **MV_CalcVolume**: Populate all volume tables and clipping table
- **MV_CalcPanTable**: Pre-calculate stereo pan values for all angle/distance combinations
- **MV_GetVoice**: Find a voice by handle in VoiceList
- **MV_VoicePlaying**: Check if a voice handle is active
- **MV_VoicesPlaying**: Count active voices
- **MV_VoiceAvailable**: Check if a voice can be allocated at a given priority
- **MV_SetVoiceMixMode**: Select appropriate mix function based on format and configuration
- **MV_TestPlayback**: Verify DMA/IRQ are working by waiting for MV_MixPage to advance
- **MV_ServiceGus / MV_ServiceRightGus**: GUS-specific buffer servicers (demand-feed model)
- **MV_ServiceRecord**: Recording buffer callback
- **MV_StartRecording / MV_StopRecord**: Recording mode setup/teardown
- **MV_LockMemory / MV_UnlockMemory**: DPMI memory locking for real-time safety

## Control Flow Notes

**Initialization → Playback Loop → Shutdown**

1. **MV_Init** (once):
   - Allocate voice pool and DMA buffers in DOS memory
   - Initialize sound card hardware
   - Pre-calculate volume and pan lookup tables
   - Start DMA playback engine
   - Test that DMA/IRQ are working

2. **Playback Loop** (per buffer, interrupt-driven):
   - Hardware DMA fires interrupt at buffer boundary
   - Interrupt calls MV_ServiceVoc()
   - MV_ServiceVoc reads DMA position, advances MV_MixPage
   - Apply reverb (if enabled) to mix buffer
   - For each voice in VoiceList, call MV_MixFunction (→ MV_Mix)
   - MV_Mix calls voice→GetSound to fetch blocks, calls voice→mix to apply pitch/volume/pan
   - If voice finishes (GetSound returns NoMoreData), remove from list, fire user callback
   - DMA hardware continues with next buffer

3. **Application API**:
   - MV_PlayXxx() to start a voice (raw, WAV, VOC, or demand-fed)
   - MV_SetVoiceVolume, MV_SetVoicePitch, MV_SetPan for real-time control
   - MV_Kill to stop a voice
   - MV_SetReverb, MV_SetVolume for global effects

4. **Shutdown** (once):
   - Stop all voices, fire callbacks
   - Stop DMA playback engine
   - Shut down sound card hardware
   - Free all memory, unlock code/data from physical RAM

## External Dependencies

**System / Low-Level**:
- `<stdlib.h>`, `<string.h>`, `<dos.h>`, `<time.h>`, `<conio.h>` — Standard C and DOS
- **dpmi.h** — DOS Protected Mode Interface (memory locking, DOS memory allocation)
- **usrhooks.h** — User-controlled memory allocation hooks
- **interrup.h** — CPU interrupt enable/disable
- **dma.h** — DMA controller access (get current position)

**Linked List Support**:
- **linklist.h** — Linked list macros (LL_SortedInsertion, LL_Add, LL_Remove, etc.)

**Sound Card Drivers** (multiple cards supported):
- **sndcards.h** — Sound card type enums (SoundBlaster, UltraSound, ProAudioSpectrum, etc.)
- **blaster.h** — Sound Blaster driver (BLASTER_Init, BLASTER_BeginBufferedPlayback, BLASTER_SetMixMode, BLASTER_DMAChannel, BLASTER_GetPlaybackRate, BLASTER_GetCurrentPos, BLASTER_StopPlayback)
- **guswave.h** — Gravis UltraSound driver (GUSWAVE_Init, GUSWAVE_StartDemandFeedPlayback, GUSWAVE_KillAllVoices, GUSWAVE_Shutdown)
- **sndscape.h** — Ensoniq SoundScape driver (SOUNDSCAPE_Init, SOUNDSCAPE_BeginBufferedPlayback, SOUNDSCAPE_SetMixMode, SOUNDSCAPE_DMAChannel, SOUNDSCAPE_StopPlayback, SOUNDSCAPE_GetCurrentPos)
- **pas16.h** — Pro Audio Spectrum driver (PAS_Init, PAS_BeginBufferedPlayback, PAS_SetMixMode, PAS_DMAChannel, PAS_GetPlaybackRate, PAS_StopPlayback, PAS_GetCurrentPos)
- **sndsrc.h** — Sound Source driver (SS_Init, SS_BeginBufferedPlayback, SS_SetMixMode, SS_SampleRate, SS_StopPlayback)

**Audio Processing**:
- **pitch.h** — Pitch/frequency scaling (PITCH_GetScale, PITCH_LockMemory, PITCH_UnlockMemory)
- **multivoc.h**, **_multivc.h** — Local header (VoiceNode definition, MV_Mix8BitMono, MV_Mix8BitStereo, MV_Mix16BitMono, MV_Mix16BitStereo variants, MV_8BitReverb, MV_16BitReverb, MV_8BitReverbFast, MV_16BitReverbFast, ClearBuffer_DW)

**Debug Output**:
- **debugio.h** — Debug output functions

---

**Key Macros Defined Locally**:
- `RoundFixed(fixedval, bits)` — Fixed-point rounding
- `IS_QUIET(ptr)` — Check if volume pointer is zero (silent)
- `MV_SetErrorCode(status)` — Update error code
- `MV_LockStart` / `MV_LockEnd` — Memory-lock region markers for MV_Mix and related functions
