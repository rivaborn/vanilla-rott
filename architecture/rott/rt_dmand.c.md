# rott/rt_dmand.c

## File Purpose
Manages demand-fed streaming audio for both playback (remote voice transmission) and recording (voice capture for remote transmission) in networked gameplay. Implements circular buffer management to handle audio chunks efficiently, with callback-based integration to the FX (sound effects) subsystem.

## Core Responsibilities
- Initialize and manage playback/recording buffers for network audio transmission
- Handle circular buffer pointer management for streaming audio chunks
- Provide callback integration points for the audio subsystem (`SD_UpdatePlaybackSound`, `SD_UpdateRecordingSound`)
- Retrieve recorded audio data for network transmission via `SD_GetSoundData`
- Synchronize playback/recording state with network and audio device state
- Manage semaphore flags for cross-system recording activation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `recordstate` | enum | Return status for `SD_GetSoundData`: nodata, newsound, endsound, data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Recording` | boolean | static | Indicates if audio recording is currently active |
| `Playback` | boolean | static | Indicates if playback reception mode is active |
| `Playing` | boolean | static | Indicates if playback is actually producing audio |
| `Feeder` | boolean | static | Indicates if recorded data is available for transmission |
| `RecordingBuffer` | byte* | static | Circular buffer for recorded audio data |
| `PlaybackBuffer` | byte* | static | Circular buffer for incoming playback audio |
| `Playingvoice` | int | static | Voice handle returned by FX subsystem |
| `RecordingPointer` | int | static | Write pointer in recording circular buffer |
| `FeederPointer` | int | static | Read pointer for retrieving recorded data |
| `PlaybackPointer` | int | static | Write pointer in playback circular buffer |
| `PlayingPointer` | int | static | Read/playback pointer in playback buffer; -1 = not started |
| `RecordingSemaphore` | boolean | static | Cross-system flag indicating recording should be active |
| `whereami` | int | extern | Debug location counter for crash analysis |

## Key Functions / Methods

### SD_StartIncomingSound
- Signature: `void SD_StartIncomingSound(void)`
- Purpose: Initialize playback mode to receive streamed audio from network
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates `PlaybackBuffer`, sets `Playback=true`, calls `FX_StartDemandFeedPlayback` to register callback
- Calls: `SafeMalloc`, `FX_StartDemandFeedPlayback`
- Notes: Fails silently if `SD_Started==false` or if playback/recording already active; resets pointers to -1 and 0

### SD_StopIncomingSound
- Signature: `void SD_StopIncomingSound(void)`
- Purpose: Stop receiving incoming playback audio
- Inputs: None
- Outputs/Return: None
- Side effects: Sets `Playback=false`; does not deallocate buffer (cleanup occurs in `SD_UpdatePlaybackSound`)
- Calls: None
- Notes: Deferred cleanup pattern—buffer freed when playback completes naturally

### SD_UpdateIncomingSound
- Signature: `void SD_UpdateIncomingSound(byte * ptr, word length)`
- Purpose: Receive a chunk of incoming playback audio and write to circular buffer
- Inputs: `ptr` = audio chunk, `length` = chunk size in bytes
- Outputs/Return: None
- Side effects: Writes to `PlaybackBuffer` at `PlaybackPointer`, advances pointer with wrap-around; sets `Playing=true` when first data arrives; sets `Playback=false` when buffer becomes full
- Calls: `memcpy` (twice if wrap-around)
- Notes: Handles circular buffer wrap-around by splitting copy into two `memcpy` calls; uses modulo via `& (BUFFERSIZE - 1)`; signals playback start via `PlayingPointer=0` transition from -1

### SD_UpdatePlaybackSound
- Signature: `void SD_UpdatePlaybackSound(char ** ptr, unsigned long * length)`
- Purpose: Callback invoked by FX subsystem to retrieve next playback chunk
- Inputs: None (outputs via pointers)
- Outputs/Return: Sets `*ptr` to next audio chunk address and `*length` to chunk size; NULL/0 if no data available
- Side effects: Advances `PlayingPointer` with wrap-around; stops voice and deallocates buffer when playback complete and not receiving new data
- Calls: `FX_StopSound`, `SafeFree`
- Notes: Returns NULL if `Playing=false` or pointers match; size fixed to `PLAYBACKDELTASIZE`; uses -1 sentinel to indicate not-yet-started

### SD_StartRecordingSound
- Signature: `boolean SD_StartRecordingSound(void)`
- Purpose: Begin recording audio from microphone/input device
- Inputs: None
- Outputs/Return: `true` if recording started, `false` on failure
- Side effects: Allocates `RecordingBuffer`, sets `Recording=true`, registers `SD_UpdateRecordingSound` callback with FX subsystem
- Calls: `SafeMalloc`, `FX_StartRecording`
- Notes: Requires `SD_Started==true` and `remoteridicule==true`; fails if already recording/playing; `Feeder` and `FeederPointer` initialized to false/-1

### SD_UpdateRecordingSound
- Signature: `void SD_UpdateRecordingSound(char * ptr, int length)`
- Purpose: Callback invoked by FX subsystem when audio has been recorded
- Inputs: `ptr` = recorded audio chunk, `length` = size in bytes
- Outputs/Return: None
- Side effects: Writes to `RecordingBuffer` at `RecordingPointer`, advances pointer with wrap-around; sets `Feeder=true`; stops recording when write pointer catches read pointer
- Calls: `memcpy` (twice if wrap-around)
- Notes: Includes `whereami` debug markers (69–74) for crash diagnostics; handles circular buffer wrap similar to playback; sets `Recording=false` when buffer is full

### SD_GetSoundData
- Signature: `recordstate SD_GetSoundData(byte * data, word length)`
- Purpose: Retrieve next chunk of recorded audio for network transmission
- Inputs: `data` = output buffer, `length` = requested chunk size
- Outputs/Return: Returns `recordstate`: `rs_newsound` (first chunk), `rs_data` (subsequent), `rs_endsound` (finished), `rs_nodata` (no data ready)
- Side effects: Reads from `RecordingBuffer` at `FeederPointer`, advances pointer with wrap-around; deallocates buffer when recording ends and all data consumed; sets `Feeder=false`
- Calls: `memcpy` (twice if wrap-around), `SafeFree`
- Notes: Distinguishes first chunk via `FeederPointer==-1` initialization; blocking read—returns `rs_nodata` if write pointer equals read pointer and still recording; uses same circular buffer wrap pattern

### SD_SoundDataReady
- Signature: `boolean SD_SoundDataReady(void)`
- Purpose: Check if recorded audio is available for transmission
- Inputs: None
- Outputs/Return: `true` if `Feeder==true`, `false` otherwise
- Side effects: None
- Calls: None
- Notes: Simple status check; does not consume data

### SD_SetRecordingActive, SD_ClearRecordingActive
- Signature: `void SD_SetRecordingActive(void)`, `void SD_ClearRecordingActive(void)`
- Purpose: Set/clear semaphore for cross-system recording state
- Inputs: None
- Outputs/Return: None
- Side effects: Set/clear `RecordingSemaphore` flag
- Calls: None
- Notes: Allows external systems (network/UI) to signal intent to record without immediate device activation

### SD_RecordingActive
- Signature: `boolean SD_RecordingActive(void)`
- Purpose: Check if recording is flagged as active across systems
- Inputs: None
- Outputs/Return: `true` if `RecordingSemaphore==true`
- Side effects: None
- Calls: None
- Notes: Status query only

## Control Flow Notes
This module operates asynchronously via callbacks:
- **Playback**: Application calls `SD_StartIncomingSound` → network layer calls `SD_UpdateIncomingSound` with chunks → FX subsystem calls `SD_UpdatePlaybackSound` to feed audio → cleanup on completion.
- **Recording**: Application calls `SD_StartRecordingSound` → FX subsystem calls `SD_UpdateRecordingSound` as data is captured → application calls `SD_GetSoundData` to retrieve chunks for transmission → cleanup when recording ends.
- Not part of main frame loop; driven by audio device and network events.

## External Dependencies
- **FX subsystem**: `FX_StartDemandFeedPlayback`, `FX_StopSound`, `FX_StartRecording`, `FX_StopRecord` (defined elsewhere)
- **Memory**: `SafeMalloc`, `SafeFree` (rt_util.h)
- **Global state**: `SD_Started` (rt_sound.h), `remoteridicule` (rt_net.h)
- **Debug**: `whereami` extern (develop.h)
- **Headers**: Includes rt_def.h (types), rt_util.h, rt_sound.h, rt_net.h, _rt_dman.h, fx_man.h, develop.h, memcheck.h
