# audiolib/source/mv1.c

## File Purpose
Core multichannel digitized sound mixer for Sound Blaster and compatible cards. Implements real-time audio mixing of multiple voices (channels) into a double-buffered output, with support for mono/stereo and 8-bit/16-bit formats. Handles voice allocation, 3D positioning, pitch/pan control, and VOC format playback.

## Core Responsibilities
- **Voice Management**: Allocate/deallocate voice nodes from a pool, with priority-based voice stealing for overcommitment
- **Audio Mixing**: Mix multiple voices into a mono or stereo buffer at 8-bit or 16-bit resolution using fixed-point rate scaling
- **Playback Control**: Manage double-buffering scheme, interrupt-driven buffer service, and integration with hardware sound cards
- **Spatial Audio**: 3D panning and distance attenuation via lookup tables; pitch scaling via external pitch module
- **Format Handling**: Parse VOC format sound blocks, manage looping, and detect format variants
- **Volume & Clipping**: Precalculate volume lookup tables per voice, apply soft clipping via harsh clip tables

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| VoiceNode | struct (defined elsewhere) | Represents a single voice/channel with position, volume, playback state |
| VList | struct (defined elsewhere) | Doubly-linked list container (start/end pointers) for voice lists |
| Pan | struct (defined elsewhere) | Stereo pan information (left/right volume pair) |
| VOLUME_TABLE_8BIT | typedef (defined elsewhere) | 256-entry lookup table for 8-bit volume scaling |
| VOLUME_TABLE_16BIT | typedef (defined elsewhere) | 256-entry lookup table for 16-bit volume scaling |
| MONO8, STEREO8 | typedef (defined elsewhere) | 8-bit mono/stereo sample types |
| MONO16, STEREO16 | typedef (defined elsewhere) | 16-bit mono/stereo sample types |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MV_VolumeTable | signed char[] | static | Master volume lookup table backing both 8-bit and 16-bit tables |
| MV_8BitVolumeTable | VOLUME_TABLE_8BIT* | static | Points into MV_VolumeTable; maps (voice volume, sample byte) → output byte |
| MV_16BitVolumeTable | VOLUME_TABLE_16BIT* | static | Points into MV_VolumeTable; maps (voice volume, sample byte) → output word |
| MV_PanTable | Pan[MV_NumPanPositions][MV_MaxVolume+1] | static | Stereo pan levels by angle and distance |
| MV_Installed | int | static | Boolean flag; TRUE when initialized, used as guard for public API |
| MV_SoundCard | int | static | Card type (SoundBlaster, Awe32, ProAudioSpectrum, etc.) |
| MV_TotalVolume | int | static | Master output volume (0–MV_MaxTotalVolume) |
| MV_MaxVoices | int | static | Maximum allocated voices |
| MV_MixMode | int | static | Current mix format (MONO_8BIT, STEREO_8BIT, MONO_16BIT, STEREO_16BIT) |
| MV_Silence | int | static | Fill value for buffer clear (0x00 for 8-bit, 0x8000 for 16-bit) |
| MV_BufferSize | int | static | Size in bytes of one mix buffer |
| MV_SampleSize | int | static | Bytes per sample (1 for MONO8, 2 for STEREO8 or MONO16, 4 for STEREO16) |
| MV_NumberOfBuffers | int | static | Count of buffers in ring (typically 2–4 for double/triple buffering) |
| MV_MixRate | int | static | Actual playback sample rate (set by sound card driver) |
| MV_RequestedMixRate | int | static | Requested sample rate at init |
| MV_SwapLeftRight | int | static | Boolean; TRUE for SBPro (reverses panning) |
| MV_BufferDescriptor | int | static | DOS memory descriptor from DPMI_GetDOSMemory |
| MV_MixBuffer | char*[NumberOfBuffers] | static | Array of pointers to DOS-allocated mix buffers |
| MV_Voices | VoiceNode* | static | Dynamically allocated voice pool |
| VoiceList | volatile VList | static | Doubly-linked list of active playing voices |
| VoicePool | volatile VList | static | Doubly-linked list of free/available voices |
| MV_MixPage | int | static | Index of buffer currently being mixed (0–MV_NumberOfBuffers-1) |
| MV_PlayPage | int | static | Index of buffer currently being played |
| MV_VoiceHandle | int | static | Running counter for allocating unique voice handles |
| MV_CallBackFunc | void(*)(unsigned long) | static | Optional callback invoked when a voice finishes; passed voice callbackval |
| HarshClipTable | char* | global | Lookup table for 8-bit clipping; accessed during mixing (memory-locked) |
| HarshClipTable16 | unsigned short* | global | Lookup table for 16-bit clipping; casts same allocation as HarshClipTable |
| HarshClipTable16s | short* | global | Signed variant of HarshClipTable16 (unused in current code) |
| MV_ErrorCode | int | global | Last error status set via MV_SetErrorCode macro |

## Key Functions / Methods

### MV_Init
- **Signature**: `int MV_Init(int soundcard, int MixRate, int Voices, int MixMode)`
- **Purpose**: Initialize Multivoc system, allocate resources, set up sound card, and start playback engine.
- **Inputs**:
  - `soundcard`: Card type (SoundBlaster, Awe32, ProAudioSpectrum, SoundMan16, SoundSource, TandySoundSource)
  - `MixRate`: Desired playback sample rate (e.g., 11025, 22050, 44100)
  - `Voices`: Maximum concurrent voices
  - `MixMode`: Format (MONO_8BIT, STEREO_8BIT, MONO_16BIT, STEREO_16BIT)
- **Outputs/Return**: MV_Ok on success; MV_Error on failure (sets MV_ErrorCode)
- **Side effects**: Calls MV_Shutdown if already installed; locks memory; allocates DOS memory via DPMI; initializes sound card driver; starts interrupt-driven playback
- **Calls**: MV_Shutdown, MV_LockMemory, USRHOOKS_GetMem, DPMI_GetDOSMemory, BLASTER_Init/PAS_Init/SS_Init (card-specific), MV_SetMixMode, MV_CalcPanTable, MV_SetVolume, PITCH_Init, MV_StartPlayback
- **Notes**: On error, releases resources and restores error code before returning. Uses interrupt-based mixing; all mixing code must be memory-locked.

### MV_Play
- **Signature**: `int MV_Play(char *ptr, int pitchoffset, int vol, int left, int right, int priority, unsigned long callbackval)`
- **Purpose**: Start playback of a VOC-format sound at specified volume, pan, and priority.
- **Inputs**:
  - `ptr`: Pointer to VOC file data (in DOS memory)
  - `pitchoffset`: Pitch adjustment (used to calculate RateScale via PITCH_GetScale)
  - `vol`: Master volume (0–255)
  - `left`, `right`: Stereo pan volumes (0–255)
  - `priority`: Priority for voice stealing; higher priority steals lower priority voices
  - `callbackval`: Opaque value passed to callback when voice finishes
- **Outputs/Return**: Voice handle (positive int) on success; MV_Error on failure (sets MV_ErrorCode)
- **Side effects**: Allocates voice from pool (may steal lower-priority voice); parses first VOC block; adds voice to VoiceList; marks all buffers as active for this voice
- **Calls**: MV_AllocVoice, PITCH_GetScale, MV_GetNextVOCBlock, LL_AddToTail (linked list macro), DisableInterrupts, RestoreInterrupts
- **Notes**: VOC header offset 0x14 gives start of first block. Voice inserted into list with interrupts disabled to avoid race with MV_ServiceVoc.

### MV_Play3D
- **Signature**: `int MV_Play3D(char *ptr, int pitchoffset, int angle, int distance, int priority, unsigned long callbackval)`
- **Purpose**: Wrapper for MV_Play that calculates stereo pan and volume from 3D angle/distance via lookup table.
- **Inputs**: Same as MV_Play except `angle` (0–31, angle around listener) and `distance` (0–255, distance from listener) replace left/right/vol
- **Outputs/Return**: Voice handle or MV_Error
- **Side effects**: Indexes MV_PanTable to compute left/right/mid volumes before calling MV_Play
- **Calls**: MV_Play, max (macro)
- **Notes**: Negative distance flips angle by MV_NumPanPositions/2 (behind listener). Mid volume = 255 - distance.

### MV_ServiceVoc
- **Signature**: `void MV_ServiceVoc(void)`
- **Purpose**: Interrupt handler called by sound card driver on each buffer boundary; manages double buffering and prepares next mix buffer.
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Swaps MV_MixPage and MV_PlayPage, deletes dead voices from current play page, calls MV_PrepareBuffer on next mix page
- **Calls**: MV_DeleteDeadVoices, MV_PrepareBuffer
- **Notes**: Invoked as callback from BLASTER_BeginBufferedPlayback or equivalent. Critical real-time function; must complete quickly and safely with interrupts disabled.

### MV_PrepareBuffer
- **Signature**: `void MV_PrepareBuffer(int page)`
- **Purpose**: Clear mix buffer and blend all active voices into it.
- **Inputs**: `page`: Buffer index to prepare (0–MV_NumberOfBuffers-1)
- **Outputs/Return**: None
- **Side effects**: Clears buffer to MV_Silence; iterates VoiceList and calls appropriate mix function (MV_Mix8bitMono, MV_Mix8bitStereo, MV_Mix16bitUnsignedMono, or MV_Mix16bitUnsignedStereo) for each voice; sets voice->Active[page] based on Playing flag
- **Calls**: ClearBuffer_DW (clears 32-bit words), MV_Mix8bitMono/Stereo/16bitUnsignedMono/Stereo
- **Notes**: Must be memory-locked and interrupt-safe. Called from MV_ServiceVoc.

### MV_Mix8bitMono
- **Signature**: `void MV_Mix8bitMono(VoiceNode *voice, int buffer)`
- **Purpose**: Accumulate a voice into an 8-bit mono mix buffer using fixed-point playback rate, volume lookup, and harsh clipping.
- **Inputs**:
  - `voice`: Voice node with position (16.16 fixed-point), sound pointer, sampling rate, length
  - `buffer`: Mix buffer index
- **Outputs/Return**: None
- **Side effects**: Updates voice->position; calls MV_GetNextVOCBlock if position exceeds voice->length; reads from HarshClipTable
- **Calls**: MV_GetNextVOCBlock, indirectly accesses MV_8BitVolumeTable and HarshClipTable (both memory-locked)
- **Notes**: Position is 16.16 fixed-point (integer position in upper 16 bits); incremented by voice->RateScale per sample. Clipping done via lookup table indexed by (4*256 + accumulated_sample - 0x80), avoiding saturation arithmetic. Memory-locked.

### MV_Mix8bitStereo
- **Signature**: `void MV_Mix8bitStereo(VoiceNode *voice, int buffer)`
- **Purpose**: Accumulate voice into 8-bit stereo buffer with separate left and right volume tables.
- **Inputs**: Same as MV_Mix8bitMono
- **Outputs/Return**: None
- **Side effects**: Updates voice->position; calls MV_GetNextVOCBlock if exhausted; updates mix buffer left/right samples
- **Calls**: MV_GetNextVOCBlock
- **Notes**: Uses voice->LeftVolume and voice->RightVolume (set by MV_SetPan or MV_Pan3D). Memory-locked.

### MV_Mix16bitUnsignedMono
- **Signature**: `void MV_Mix16bitUnsignedMono(VoiceNode *voice, int buffer)`
- **Purpose**: Accumulate voice into 16-bit mono buffer; handles unsigned 16-bit samples and 16-bit harsh clipping table.
- **Inputs**: Same as MV_Mix8bitMono
- **Outputs/Return**: None
- **Side effects**: Updates voice->position and calls MV_GetNextVOCBlock if needed
- **Calls**: MV_GetNextVOCBlock
- **Notes**: Samples treated as unsigned (0x0000–0xFFFF, centered at 0x8000). Clipping via HarshClipTable16[4*256*16 + sample_index]. Scaling operations shift by 4 and 20 bits. Memory-locked.

### MV_Mix16bitUnsignedStereo
- **Signature**: `void MV_Mix16bitUnsignedStereo(VoiceNode *voice, int buffer)`
- **Purpose**: Accumulate voice into 16-bit stereo buffer.
- **Inputs**: Same as MV_Mix8bitMono
- **Outputs/Return**: None
- **Side effects**: Updates voice->position; updates left and right 16-bit samples in mix buffer
- **Calls**: MV_GetNextVOCBlock
- **Notes**: Memory-locked. Uses voice->LeftVolume and voice->RightVolume.

### MV_GetNextVOCBlock
- **Signature**: `void MV_GetNextVOCBlock(VoiceNode *voice)`
- **Purpose**: Parse next block in VOC file format; handle block types (sound data, continuation, silence, marker, repeat, extended, new format); update voice->sound, voice->length, voice->SamplingRate, voice->RateScale.
- **Inputs**: `voice`: Voice node with NextBlock pointer positioned at block header
- **Outputs/Return**: None
- **Side effects**: Advances voice->NextBlock through VOC blocks; sets voice->Playing = TRUE/FALSE; updates voice->sound, voice->length (in 16.16 fixed-point), voice->SamplingRate, voice->RateScale, voice->LoopStart, voice->LoopCount
- **Calls**: PITCH_GetScale (indirectly via voice->PitchScale)
- **Notes**: Handles block types 0 (end), 1 (sound data), 2 (continuation), 3 (silence), 4 (marker), 5 (ASCII), 6 (repeat begin), 7 (repeat end), 8 (extended), 9 (new format). Only supports 8-bit mono PCM (block 9 with BitsPerSample=8, Channels=1, Format=0). Skips packed/stereo data. On exhaustion during playback, wraps to LoopStart if loop count > 0.

### MV_AllocVoice
- **Signature**: `VoiceNode *MV_AllocVoice(int priority)`
- **Purpose**: Retrieve an available voice or steal one with lower priority if pool empty.
- **Inputs**: `priority`: Priority of new voice; steals voices with lower priority
- **Outputs/Return**: Pointer to allocated VoiceNode; NULL if no voices available after stealing
- **Side effects**: Removes voice from VoicePool; if pool empty, calls MV_Kill on lowest-priority active voice; generates new unique handle via MV_VoiceHandle
- **Calls**: MV_Kill, MV_VoicePlaying, DisableInterrupts, RestoreInterrupts, LL_Remove
- **Notes**: Handle generation loops until MV_VoicePlaying returns FALSE for handle. Interrupts disabled during pool access.

### MV_Kill
- **Signature**: `int MV_Kill(int handle)`
- **Purpose**: Stop playback and remove a voice from the active list.
- **Inputs**: `handle`: Voice handle returned by MV_Play
- **Outputs/Return**: MV_Ok on success; MV_Error if not installed or voice not found
- **Side effects**: Moves voice from VoiceList to VoicePool; invokes MV_CallBackFunc(callbackval) if registered
- **Calls**: MV_GetVoice, LL_Remove, LL_AddToTail, DisableInterrupts, RestoreInterrupts
- **Notes**: Callback invoked outside interrupt critical section.

### MV_SetPan
- **Signature**: `int MV_SetPan(int handle, int vol, int left, int right)`
- **Purpose**: Set overall volume and stereo pan for a voice.
- **Inputs**:
  - `handle`: Voice handle
  - `vol`: Overall volume (0–255)
  - `left`: Left channel volume (0–255)
  - `right`: Right channel volume (0–255)
- **Outputs/Return**: MV_Ok on success; MV_Warning/MV_Error if voice not found
- **Side effects**: Updates voice->Volume, voice->LeftVolume, voice->RightVolume via MIX_VOLUME macro; may swap left/right if MV_SwapLeftRight TRUE (SBPro compatibility)
- **Calls**: MV_GetVoice, MIX_VOLUME (macro, defined elsewhere)
- **Notes**: Macro MIX_VOLUME scales 0–255 input to internal volume table index (typically 0–127).

### MV_Pan3D
- **Signature**: `int MV_Pan3D(int handle, int angle, int distance)`
- **Purpose**: Set 3D position of a voice (angle and distance from listener) and apply appropriate panning and attenuation.
- **Inputs**:
  - `handle`: Voice handle
  - `angle`: Angle around listener (0–31)
  - `distance`: Distance from listener (0–255)
- **Outputs/Return**: Result of MV_SetPan
- **Side effects**: Indexes MV_PanTable; calls MV_SetPan with computed left, right, mid volumes
- **Calls**: MV_SetPan, max (macro), MIX_VOLUME (macro)
- **Notes**: Negative distance flips angle; mid = 255 - distance; pan table indexed by angle and MIX_VOLUME(distance).

### MV_SetPitch
- **Signature**: `int MV_SetPitch(int handle, int pitchoffset)`
- **Purpose**: Change pitch of a playing voice without stopping/restarting it.
- **Inputs**:
  - `handle`: Voice handle
  - `pitchoffset`: Pitch offset (semantics defined by PITCH module; typically semitone units)
- **Outputs/Return**: MV_Ok on success; MV_Error if not installed or voice not found
- **Side effects**: Updates voice->PitchScale via PITCH_GetScale; recalculates voice->RateScale = (voice->SamplingRate * voice->PitchScale) / MV_MixRate
- **Calls**: MV_GetVoice, PITCH_GetScale
- **Notes**: RateScale is fixed-point playback rate increment per sample.

### MV_SetVolume
- **Signature**: `void MV_SetVolume(int volume)`
- **Purpose**: Set master output volume and recalculate all volume lookup tables.
- **Inputs**: `volume`: Master volume level (0–MV_MaxTotalVolume)
- **Outputs/Return**: None
- **Side effects**: Clamps input to [0, MV_MaxTotalVolume]; updates MV_TotalVolume; calls MV_CalcVolume
- **Calls**: MV_CalcVolume, max/min (macros)
- **Notes**: Non-real-time; safe to call from main thread.

### MV_CalcVolume
- **Signature**: `void MV_CalcVolume(int MaxLevel)`
- **Purpose**: Precalculate volume translation tables for all voice volumes; calculate harsh clipping lookup tables.
- **Inputs**: `MaxLevel`: Maximum output level (scaled from MV_MaxTotalVolume)
- **Outputs/Return**: None
- **Side effects**: Fills MV_8BitVolumeTable and MV_16BitVolumeTable with (volume, sample byte) → output mappings; fills HarshClipTable and HarshClipTable16 with soft clipping curves
- **Calls**: DisableInterrupts, RestoreInterrupts
- **Notes**: Iterates over 256 input levels per volume level (typically 128 volume levels). For 8-bit, clips to [0, 255]; for 16-bit, clips to [0, 0xFFFF]. Harsh clip table indexed by accumulated sample (with offset 4*256 or 4*256*16) to allow negative samples to clip to 0.

### MV_CalcPanTable
- **Signature**: `void MV_CalcPanTable(void)`
- **Purpose**: Precalculate stereo panning levels for all angle/distance combinations.
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Fills MV_PanTable[angle][distance] with left/right volume pairs based on angle (0–31, quarter circle) and distance
- **Calls**: None
- **Notes**: Creates smooth panning curve; HalfAngle = MV_NumPanPositions/2. Ramp calculated per quadrant; table uses symmetry to fill all angles.

### MV_StartPlayback
- **Signature**: `int MV_StartPlayback(void)`
- **Purpose**: Start the interrupt-driven sound playback engine with configured sound card.
- **Inputs**: None
- **Outputs/Return**: MV_Ok on success; MV_Error on failure (sets MV_ErrorCode)
- **Side effects**: Initializes buffers to silence; sets MV_PlayPage = 0, MV_MixPage = 1; calls card-specific BeginBufferedPlayback (passes MV_ServiceVoc as callback); sets MV_MixRate from card driver
- **Calls**: ClearBuffer_DW, BLASTER_BeginBufferedPlayback/PAS_BeginBufferedPlayback/SS_BeginBufferedPlayback (card-specific)
- **Notes**: Called during MV_Init. Card-specific functions return actual mix rate (may differ from requested).

### MV_StopPlayback
- **Signature**: `void MV_StopPlayback(void)`
- **Purpose**: Halt interrupt-driven playback and disable sound card output.
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Calls card-specific StopPlayback function
- **Calls**: BLASTER_StopPlayback/PAS_StopPlayback/SS_StopPlayback (card-specific)
- **Notes**: Called during MV_Shutdown.

### MV_Shutdown
- **Signature**: `int MV_Shutdown(void)`
- **Purpose**: Clean up and deallocate all resources allocated by MV_Init.
- **Inputs**: None
- **Outputs/Return**: MV_Ok on success; MV_Error if not installed
- **Side effects**: Kills all voices; stops playback; calls card-specific shutdown; frees allocated memory (DOS memory, voice pool); resets state variables
- **Calls**: MV_KillAllVoices, MV_StopPlayback, card-specific Shutdown functions, USRHOOKS_FreeMem, DPMI_FreeDOSMemory, DisableInterrupts, RestoreInterrupts
- **Notes**: Safe to call even if initialization failed. Sets MV_Installed = FALSE early.

### MV_LockMemory / MV_UnlockMemory
- **Signature**: `int MV_LockMemory(void)` / `void MV_UnlockMemory(void)`
- **Purpose**: Lock critical mixing code and data into physical memory to prevent page faults during interrupt handling. Unlock when done.
- **Inputs**: None
- **Outputs/Return**: MV_Ok on success; MV_Error on failure
- **Side effects**: Calls DPMI_LockMemoryRegion for mix functions (MV_Mix8bitMono through MV_LockEnd) and DPMI_Lock for each state variable; calls PITCH_LockMemory/PITCH_UnlockMemory
- **Calls**: DPMI_LockMemoryRegion, DPMI_Lock, DPMI_Unlock, PITCH_LockMemory, PITCH_UnlockMemory
- **Notes**: Locks contiguous code region and all global state accessed during mixing/interrupt handling. Must succeed for real-time safety.

### MV_ErrorString
- **Signature**: `char *MV_ErrorString(int ErrorNumber)`
- **Purpose**: Return human-readable error message for a given error code.
- **Inputs**: `ErrorNumber`: Error code (MV_Ok, MV_NotInstalled, MV_NoVoices, etc.) or -1 for current error
- **Outputs/Return**: Pointer to static string
- **Side effects**: If ErrorNumber is MV_Error or MV_Warning, recursively calls itself on MV_ErrorCode
- **Calls**: Recursive call on MV_Error/MV_Warning
- **Notes**: Returns "Unknown Multivoc error code." for unrecognized codes.

**Trivial helper functions** (not fully documented):
- `MV_GetVoice(int handle)`: Find voice by handle; returns NULL and sets error if not found
- `MV_VoicePlaying(int handle)`: Check if voice handle is active
- `MV_VoicesPlaying(void)`: Count active voices
- `MV_KillAllVoices(void)`: Stop all voices
- `MV_DeleteDeadVoices(int page)`: Remove inactive voices from VoiceList and invoke callback
- `MV_SetCallBack(void (*func)(unsigned long))`: Register voice completion callback
- `MV_GetVolume(void)`: Return current master volume
- `MV_SetMixMode(int mode)`: Configure buffer format and recalculate sizes
- `MV_StartRecording(int, void(*)(char*, int))`: Stub (returns MV_Ok, unimplemented)
- `MV_StopRecord(void)`: Stub (no-op)
- `MV_StartDemandFeedPlayback(...)`: Stub (returns MV_Ok, unimplemented)
- `MV_LockEnd(void)`: Marker function for memory locking region end

## Control Flow Notes

**Initialization path**: `MV_Init()` → memory locking → DOS memory allocation → voice pool creation → sound card initialization → buffer setup → volume/pan table precalculation → `MV_StartPlayback()`

**Real-time mixing loop** (interrupt-driven):
1. Sound card interrupt fires on buffer boundary
2. Calls `MV_ServiceVoc()`
3. `MV_DeleteDeadVoices()` removes finished voices and invokes callbacks
4. Toggles `MV_PlayPage` and `MV_MixPage`
5. `MV_PrepareBuffer()` clears next mix buffer and calls appropriate mix function for each active voice
6. Mix functions use lookup tables and harsh clipping; advance voice positions

**Voice playback**:
- `MV_Play()` or `MV_Play3D()` allocates voice and calls `MV_GetNextVOCBlock()` to parse first VOC block
- Voice added to `VoiceList` with interrupts disabled
- On each interrupt, voice mixed into current buffer
- When voice playback exhausted, removed from `VoiceList`, returned to `VoicePool`, callback invoked
- `MV_Kill()` can manually stop voice at any time

**Shutdown path**: `MV_Shutdown()` → kill all voices → stop playback → uninitialize sound card → release memory

## External Dependencies

**Direct includes / imports**:
- `stdlib.h`: Standard C library (malloc, etc.; not used directly in this file but likely in implementations)
- `dpmi.h`: DPMI (DOS Protected Mode Interface) for memory locking and DOS memory allocation
- `usrhooks.h`: User-provided memory allocation hooks
- `interrup.h`: Interrupt enable/disable macros
- `ll_man.h`: Linked list management macros
- `sndcards.h`: Sound card driver abstraction (defines SoundBlaster, Awe32, ProAudioSpectrum, etc.)
- `blaster.h`: Sound Blaster driver API
- `sndsrc.h`: Sound Source driver API (conditional)
- `pas16.h`: Pro Audio Spectrum driver API
- `pitch.h`: Pitch scaling module
- `multivoc.h`: Public header (client-facing API)
- `_multivc.h`: Private header (likely defines VoiceNode, VList, Pan, volume table types)

**External symbols used but not defined here**:
- `BLASTER_Init()`, `BLASTER_BeginBufferedPlayback()`, `BLASTER_StopPlayback()`, `BLASTER_Shutdown()`, `BLASTER_GetPlaybackRate()`, `BLASTER_SetMixMode()`, `BLASTER_Error`, `BLASTER_ErrorString()`
- `PAS_Init()`, `PAS_BeginBufferedPlayback()`, `PAS_StopPlayback()`, `PAS_Shutdown()`, `PAS_GetPlaybackRate()`, `PAS_SetMixMode()`, `PAS_Error`, `PAS_ErrorString()`
- `SS_Init()`, `SS_BeginBufferedPlayback()`, `SS_StopPlayback()`, `SS_Shutdown()`, `SS_SetMixMode()`, `SS_Error`, `SS_ErrorString()`, `SS_SampleRate`
- `ClearBuffer_DW()`: Clear buffer by 32-bit word (likely optimized memset variant)
- `PITCH_Init()`, `PITCH_GetScale()`, `PITCH_LockMemory()`, `PITCH_UnlockMemory()`: Pitch module
- `USRHOOKS_GetMem()`, `USRHOOKS_FreeMem()`: User-provided memory allocation
- `DPMI_GetDOSMemory()`, `DPMI_FreeDOSMemory()`, `DPMI_LockMemory()`, `DPMI_LockMemoryRegion()`, `DPMI_Unlock()`, `DPMI_UnlockMemory()`, `DPMI_UnlockMemoryRegion()`: DPMI interface
- `DisableInterrupts()`, `RestoreInterrupts()`: Inline interrupt control
- `LL_AddToHead()`, `LL_AddToTail()`, `LL_Remove()`: Linked list macros
- `MIX_VOLUME()`: Volume scaling macro
- `max()`, `min()`: Common macros
