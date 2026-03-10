# rott/rt_dmand.h

## File Purpose
Header file for sound demand management, providing interfaces to receive incoming sound data in chunks and manage sound recording. Part of the game engine's audio streaming subsystem (SD_ prefix indicates sound driver functions).

## Core Responsibilities
- Establish and teardown streaming audio input pipelines
- Buffer and retrieve incoming sound data in discrete chunks
- Query audio data availability and stream state
- Manage sound recording state (active/inactive)
- Coordinate recording start/stop and data flow

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| recordstate | enum | State transitions for incoming sound: nodata, newsound (start of new stream), data (chunk ready), endsound (stream end) |

## Global / File-Static State
None.

## Key Functions / Methods

### SD_StartIncomingSound
- Signature: `void SD_StartIncomingSound(void)`
- Purpose: Initialize the audio reception pipeline for streaming incoming sound data
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up internal buffering for incoming audio chunks
- Calls: Not inferable from this file

### SD_StopIncomingSound
- Signature: `void SD_StopIncomingSound(void)`
- Purpose: Halt audio reception and playback of incoming sound
- Inputs: None
- Outputs/Return: None
- Side effects: Flushes/closes audio reception pipeline
- Calls: Not inferable from this file

### SD_UpdateIncomingSound
- Signature: `void SD_UpdateIncomingSound(byte *data, word length)`
- Purpose: Feed next chunk of incoming audio data into the buffer
- Inputs: `data` (audio chunk), `length` (byte count)
- Outputs/Return: None
- Side effects: Updates internal audio buffer
- Calls: Not inferable from this file

### SD_GetSoundData
- Signature: `recordstate SD_GetSoundData(byte *data, word length)`
- Purpose: Retrieve buffered sound data and query stream state
- Inputs: `data` (output buffer), `length` (max bytes to retrieve)
- Outputs/Return: recordstate enum indicating stream condition (nodata/newsound/data/endsound)
- Side effects: Consumes data from internal buffer
- Calls: Not inferable from this file
- Notes: Returns `newsound` only at stream start; caller determines readiness via returned state

### SD_SoundDataReady
- Signature: `boolean SD_SoundDataReady(void)`
- Purpose: Check if audio data is available for retrieval
- Inputs: None
- Outputs/Return: Boolean flag
- Calls: Not inferable from this file

### SD_SetRecordingActive / SD_ClearRecordingActive
- Purpose: Enable/disable recording state flag
- Side effects: Toggles internal recording flag (platform-specific implementation)
- Notes: State queries performed via `SD_RecordingActive()`

### SD_RecordingActive
- Signature: `boolean SD_RecordingActive(void)`
- Purpose: Query whether sound recording is active
- Outputs/Return: Boolean flag
- Calls: Not inferable from this file

### SD_StartRecordingSound / SD_StopRecordingSound
- Purpose: Initiate and terminate sound recording capture
- Signature (start): `boolean SD_StartRecordingSound(void)` — returns success/failure
- Signature (stop): `void SD_StopRecordingSound(void)`
- Side effects: Allocates/deallocates recording buffers; I/O to recording destination
- Calls: Not inferable from this file

## Control Flow Notes
This module operates as a demand-driven audio pipeline: incoming audio arrives asynchronously and is buffered, while the game pulls audio data on-demand via `SD_GetSoundData()`. Recording is orthogonal (separate state management). Typical frame flow: `SD_UpdateIncomingSound()` called by driver, game calls `SD_SoundDataReady()` and `SD_GetSoundData()` to consume buffered chunks.

## External Dependencies
- Primitive types: `byte`, `word`, `boolean` (defined elsewhere; typical C89 typedefs for DOS/early platform compatibility)
- No external includes visible in this header
