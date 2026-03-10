# audiolib/source/_multivc.h

## File Purpose
Private header for the MULTIVOC audio mixing library. Defines structures, constants, and internal function declarations for multi-voice audio playback, mixing, and format support (VOC, WAV, raw). Implements audio mixing for different sample formats and stereo/mono configurations.

## Core Responsibilities
- Define voice node structure and doubly-linked list for active voices
- Declare audio format constants (8-bit, 16-bit, ADPCM, A-law, µ-law)
- Define WAV/VOC file format structures
- Declare internal mixing functions for different bit depths and channels
- Provide volume and pan lookup table infrastructure
- Declare voice lifecycle management (allocation, playback control, servicing)
- Define audio reverberation routines

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `VoiceNode` | struct | Represents a single active voice; contains playback state (position, rate, volume), audio data pointers, callbacks, and function pointers for format-specific mixing |
| `VList` | struct | Doubly-linked list node for managing active voices |
| `Pan` | struct | Left/right pan position (char each) |
| `wavedata` | enum | Audio source format: Raw, VOC, DemandFeed, WAV |
| `playbackstatus` | enum | Voice playback state: NoMoreData or KeepPlaying |
| `STEREO16` / `STEREO8` | struct | Stereo sample containers (16-bit and 8-bit signed) |
| `SIGNEDSTEREO16` | struct | Signed stereo 16-bit samples |
| `riff_header` | struct | WAV file RIFF header ("RIFF"/"WAVE" markers, format chunk) |
| `format_header` | struct | WAV format descriptor (channels, sample rate, bits per sample) |
| `data_header` | struct | WAV data chunk header |
| `VOLUME8` / `VOLUME16` | typedef array | 256-entry volume lookup tables (8-bit and 16-bit) |
| `HARSH_CLIP_TABLE_8` | typedef | Clipping table for 8 voices × 256 samples |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MV_MaxPanPosition` | #define | macro | Maximum pan position index (31) |
| `MV_NumVoices` | #define | macro | Maximum concurrent voices (8) |
| `MixBufferSize` | #define | macro | Mixing buffer size in samples (256) |
| `TotalBufferSize` | #define | macro | Total ring buffer size (256 × 16) |
| `SILENCE_16BIT` / `SILENCE_8BIT` | #define | macro | Silence patterns for clearing buffers |

## Key Functions / Methods

### MV_Mix
- Signature: `static void MV_Mix(VoiceNode *voice, int buffer)`
- Purpose: Mix a single voice into the output buffer
- Inputs: Voice node, buffer index
- Outputs/Return: None
- Side effects: Writes to output buffer; updates voice position
- Calls: Voice's mixed function pointer (`voice->mix`)
- Notes: Core mixing loop; dispatches to format-specific mixers

### MV_PlayVoice / MV_StopVoice
- Signature: `static void MV_PlayVoice(VoiceNode *voice)` / `static void MV_StopVoice(VoiceNode *voice)`
- Purpose: Start/stop playback of a voice
- Inputs: Voice node
- Outputs/Return: None
- Side effects: Modifies voice state and linked list
- Notes: PlayVoice likely inserts into active list; StopVoice removes

### MV_GetNextVOCBlock / MV_GetNextDemandFeedBlock / MV_GetNextRawBlock / MV_GetNextWAVBlock
- Signature: `static playbackstatus MV_GetNext*Block(VoiceNode *voice)`
- Purpose: Fetch next audio block for format-specific playback
- Inputs: Voice node
- Outputs/Return: Playback status (NoMoreData or KeepPlaying)
- Side effects: Updates voice->NextBlock and length
- Notes: Different implementations for VOC, WAV, raw, and demand-feed sources

### MV_AllocVoice / MV_GetVoice
- Signature: `static VoiceNode *MV_AllocVoice(int priority)` / `static VoiceNode *MV_GetVoice(int handle)`
- Purpose: Allocate a new voice by priority; retrieve voice by handle
- Inputs: Priority (or handle)
- Outputs/Return: Allocated/found VoiceNode pointer
- Side effects: Modifies voice list on allocation
- Notes: Likely evicts lower-priority voices if all 8 are in use

### MV_GetVolumeTable
- Signature: `static short *MV_GetVolumeTable(int vol)`
- Purpose: Retrieve 256-entry volume lookup table for a given volume level
- Inputs: Volume (0–255)
- Outputs/Return: Pointer to VOLUME16 lookup table
- Notes: Used for fast per-sample volume scaling

### MV_SetVoiceMixMode
- Signature: `static void MV_SetVoiceMixMode(VoiceNode *voice)`
- Purpose: Set voice's format-specific mixing function pointer
- Inputs: Voice node
- Outputs/Return: None
- Side effects: Sets `voice->mix` function pointer based on bit depth and channels
- Notes: Dispatches to MV_Mix8BitMono, MV_Mix16BitStereo, etc.

### MV_SetVoicePitch
- Signature: `static void MV_SetVoicePitch(VoiceNode *voice, unsigned long rate, int pitchoffset)`
- Purpose: Configure pitch (sampling rate) for a voice
- Inputs: Voice, target sample rate, pitch offset
- Outputs/Return: None
- Side effects: Updates `voice->RateScale` and `voice->PitchScale`
- Notes: Enables pitch shifting without re-sampling the source

### MV_CalcVolume / MV_CalcPanTable
- Signature: `static void MV_CalcVolume(int MaxLevel)` / `static void MV_CalcPanTable(void)`
- Purpose: Pre-calculate volume and pan lookup tables
- Inputs: Max volume level (or none for pan table)
- Outputs/Return: None
- Side effects: Populates global VOLUME8/VOLUME16 and pan tables
- Notes: Called during initialization to avoid per-sample calculations

### MV_Mix8BitMono / MV_Mix16BitStereo (and 6 variants)
- Signature: `void MV_Mix*Bit*(unsigned long position, unsigned long rate, char *start, unsigned long length)`
- Purpose: Mix audio samples of specific format into output buffer
- Inputs: Sample position, playback rate, audio data start, length
- Outputs/Return: None
- Side effects: Updates output mix buffer; moves sample position
- Calls: Volume lookup tables
- Notes: Compiled directly from inline assembly or C; one variant per format combination

### MV_16BitReverb / MV_8BitReverb / Fast variants
- Signature: `void MV_*BitReverb*(char *src, char *dest, VOLUME16 *volume, int count)` (with `#pragma aux` for assembly)
- Purpose: Apply reverberation effect to audio samples
- Inputs: Source buffer, destination, volume table, sample count
- Outputs/Return: None
- Side effects: Writes to destination buffer
- Notes: Implemented in assembly for performance; Fast variant uses bit shift instead of lookup table

### ClearBuffer_DW
- Signature: `void ClearBuffer_DW(void *ptr, unsigned data, int length)` with inline assembly
- Purpose: Clear (or fill) a buffer with a 32-bit word using `rep stosd`
- Inputs: Destination pointer (edi), fill value (eax), word count (ecx)
- Outputs/Return: None
- Side effects: Direct memory write via assembly
- Notes: Pure assembly routine; used for fast buffer initialization

## Control Flow Notes
This header defines the internal interface for audio mixing. At runtime, the main game loop would repeatedly call `MV_ServiceVoc()` (not declared here but referenced) and `MV_Mix()` for each active voice. Voice allocation happens when sounds are requested; mixing happens continuously; voices are freed when playback ends or higher-priority sounds preempt them. The reverb functions would be called on specific voices or globally for effect processing.

## External Dependencies
- VGA graphics interface: `ATR_INDEX`, `STATUS_REGISTER_1`, `inp()`, `outp()` macros for border color modification (DOS-era hardware)
- No obvious external includes; assumes MULTIVOC.C defines the implementations
- Inline assembly (`#pragma aux`) assumes x86 instruction set
