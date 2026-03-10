# audiolib/source/_guswave.h

## File Purpose

Private header file for GUS (Gravis Ultrasound) wave audio playback. Defines internal data structures and constants for managing voice channels, sound buffers, and hardware interaction in the audio subsystem.

## Core Responsibilities

- Defines playback state structures (VoiceNode, voice lists, status tracking)
- Declares WAV/VOC format parsing structures (RIFF, format, data chunks)
- Provides configuration constants (voice limits, buffer sizes, encoding types)
- Declares voice lifecycle functions (allocation, playback, format reading)
- Manages voice priority queuing and resource limits for concurrent playback

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `wavedata` | enum | Audio format type selector (Raw, VOC, DemandFeed, WAV) |
| `playbackstatus` | enum | Playback state indicator (NoMoreData, KeepPlaying, SoundDone) |
| `VoiceNode` | volatile struct | Active voice/channel state (linked-list node with audio buffers, hardware handles, callbacks) |
| `voicelist` | struct | Doubly-linked voice queue (start/end pointers) |
| `voicestatus` | volatile struct | Voice query result (voice pointer + playing flag) |
| `riff_header` | struct | WAV file container header (RIFF/WAVE markers, fmt chunk ref) |
| `format_header` | struct | WAV format chunk (channels, sample rate, bits per sample) |
| `data_header` | struct | WAV data chunk header (DATA marker + size) |

## Global / File-Static State

None.

## Key Functions / Methods

### GUSWAVE_AllocVoice
- Signature: `VoiceNode *GUSWAVE_AllocVoice( int priority )`
- Purpose: Allocate a new voice/channel from the pool
- Inputs: Priority level (higher = more important)
- Outputs/Return: Pointer to allocated VoiceNode, or NULL if exhausted
- Side effects: Modifies voice pool state
- Calls: (defined elsewhere in GUSWAVE.C)
- Notes: Respects VOICES and MAX_VOICES limits; priority determines eviction on exhaustion

### GUSWAVE_Play
- Signature: `int GUSWAVE_Play( VoiceNode *voice, int angle, int volume, int channels )`
- Purpose: Start playback of a voice with mixing parameters
- Inputs: Voice node, stereo angle, volume (0–MAX_VOLUME=4095), channel count
- Outputs/Return: Success code
- Side effects: Updates voice state (Playing flag, Pan, Volume); triggers hardware register writes
- Calls: (defined elsewhere)
- Notes: Angle and channels configure spatial audio; voice must be allocated first

### GUSWAVE_GetVoice
- Signature: `VoiceNode *GUSWAVE_GetVoice( int handle )`
- Purpose: Look up a voice by handle
- Inputs: Voice handle identifier
- Outputs/Return: Pointer to matching VoiceNode, or NULL if not found
- Calls: (defined elsewhere)

### GUSWAVE_GetNextVOCBlock
- Signature: `playbackstatus GUSWAVE_GetNextVOCBlock( VoiceNode *voice )`
- Purpose: Fetch next audio block for VOC format playback
- Inputs: Voice currently playing
- Outputs/Return: Playback status (NoMoreData / KeepPlaying / SoundDone)
- Side effects: Updates voice->NextBlock, voice->BlockLength; may advance loop counters
- Calls: (defined elsewhere)
- Notes: Handles VOC block headers and loops; called repeatedly during playback

### GUSWAVE_InitVoices
- Signature: `static int GUSWAVE_InitVoices( void )`
- Purpose: Initialize the voice pool and GUS hardware
- Outputs/Return: Success code
- Side effects: Allocates voice structures, initializes hardware registers
- Notes: Static function; called once at engine startup

## Control Flow Notes

VoiceNode structures form a linked list managed via voicelist. Allocation (GUSWAVE_AllocVoice) assigns a node and prioritizes it. Play (GUSWAVE_Play) registers it with hardware. During frame updates, the engine repeatedly calls GetSound callback to pull audio data, and GetNextVOCBlock for VOC format streaming. The enum returns guide loop/playback control. Voices are freed when SoundDone is returned.

## External Dependencies

- No `#include` directives (private header)
- Uses `volatile` qualifier for hardware memory-mapped registers
- Function pointer callbacks (`GetSound`, `DemandFeed`) for format abstraction
- Direct hardware handle references (`GF1voice`) suggest Gravis Ultrasound driver integration
