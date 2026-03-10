# audiolib/source/guswave.c

## File Purpose

Digitized sound playback driver for the Gravis Ultrasound (GUS) audio card. Manages voice allocation, audio format parsing (VOC/WAV), real-time pitch/volume/pan control, and interrupt-driven playback callbacks on DOS-era hardware.

## Core Responsibilities

- **Voice management**: allocate/deallocate voice slots from a fixed pool, with priority-based preemption
- **Audio format support**: parse VOC and WAV file formats; support demand-fed streaming
- **Playback control**: start/stop playback, manage sample rate and bit depth, coordinate with GUS hardware
- **Pitch scaling**: apply pitch offset to playback rate via lookup table
- **3D panning**: map angle/distance to pan and volume for positional audio
- **Volume control**: master volume and per-voice volume with scaling
- **Interrupt handling**: provide callback handlers for GUS completion interrupts; maintain interrupt-safe state
- **Error reporting**: map error codes to human-readable messages

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `VoiceNode` | struct (defined elsewhere) | Represents a single voice slot with sound data, playback state, and callbacks |
| `voicelist` | struct (defined elsewhere) | Linked list header for voice chains |
| `voicestatus` | struct (defined elsewhere) | Maps GUS hardware voice number to GUSWAVE voice |
| `playbackstatus` | enum (defined elsewhere) | Playback state: `KeepPlaying`, `NoMoreData`, `SoundDone` |
| `riff_header` | struct (defined elsewhere) | WAV file RIFF/WAVE header container |
| `format_header` | struct (defined elsewhere) | WAV format chunk (sample rate, channels, bits) |
| `data_header` | struct (defined elsewhere) | WAV data chunk header |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `GUSWAVE_PanTable` | `static const int[32]` | file-static | Pan value lookup table (0–15 range, symmetric) |
| `VoiceList` | `static voicelist` | file-static | Linked list of currently active voices |
| `VoicePool` | `static voicelist` | file-static | Linked list of available (inactive) voice slots |
| `VoiceStatus` | `static voicestatus[MAX_VOICES]` | file-static | Maps GUS hardware voice index to GUSWAVE voice |
| `GUSWAVE_Voices` | `VoiceNode[VOICES]` | file-static (extern-visible) | Array of voice structures; core voice pool |
| `GUSWAVE_VoiceHandle` | `static int` | file-static | Counter for unique voice handles; starts at `GUSWAVE_MinVoiceHandle` |
| `GUSWAVE_MaxVoices` | `static int` | file-static | Number of voices successfully allocated; may be < `VOICES` if GUS memory exhausted |
| `GUSWAVE_Installed` | `int` | file-static (extern-visible) | Installation flag; `TRUE` after successful `GUSWAVE_Init()` |
| `GUSWAVE_CallBackFunc` | `static void (*)(unsigned long)` | file-static | User callback invoked when a voice completes playback |
| `GUSWAVE_Volume` | `static int` | file-static | Master volume (0–4095); used to scale all voice volumes |
| `GUSWAVE_SwapLeftRight` | `static int` | file-static | Stereo inversion flag; reverses L/R panning |
| `GUS_Debug` | `static int` | file-static | Debug mode flag; enables verbose logging and debug callback |
| `GUS_Silence8` | `static char[1024]` | file-static | Silence buffer for 8-bit audio (0x80 = silence in unsigned PCM) |
| `GUS_Silence16` | `static unsigned short[512]` | file-static | Silence buffer for 16-bit audio (0x0000 = silence in signed PCM) |
| `GUSWAVE_ErrorCode` | `int` | global | Last error code; can be queried via `GUSWAVE_ErrorString()` |

## Key Functions / Methods

### GUSWAVE_Init

- **Signature**: `int GUSWAVE_Init(int numvoices)`
- **Purpose**: Initialize the Gravis Ultrasound driver and allocate voice slots.
- **Inputs**: `numvoices` – requested number of voices (clamped to `0–VOICES`)
- **Outputs/Return**: `GUSWAVE_Ok` on success; `GUSWAVE_Error` on GUS hardware failure
- **Side effects**: 
  - Calls `GUS_Init()` to initialize hardware
  - Allocates GUS memory via `gf1_malloc()` for each voice
  - Sets `GUSWAVE_Installed = TRUE`; can trigger `GUSWAVE_Shutdown()` if already installed
- **Calls**: `GUS_Init()`, `USER_CheckParameter()`, `GUSWAVE_InitVoices()`, `GUSWAVE_SetReverseStereo()`
- **Notes**: If memory allocation fails partway, `GUSWAVE_MaxVoices` is set to the count successfully allocated. Subsequent playback is limited to this count.

### GUSWAVE_PlayVOC

- **Signature**: `int GUSWAVE_PlayVOC(char *sample, int pitchoffset, int angle, int volume, int priority, unsigned long callbackval)`
- **Purpose**: Begin playback of a VOC-format digitized sound file.
- **Inputs**: 
  - `sample` – pointer to VOC file in memory (must start with "Creative Voice File")
  - `pitchoffset` – pitch shift (passed to `PITCH_GetScale()`)
  - `angle` – pan angle (0–31, masked; mapped via `GUSWAVE_PanTable`)
  - `volume` – per-voice volume (0–255)
  - `priority` – voice priority (higher preempts lower; compared during allocation)
  - `callbackval` – opaque user data passed to completion callback
- **Outputs/Return**: Voice handle (>0) on success; `GUSWAVE_Warning` or `GUSWAVE_Error` on failure
- **Side effects**: 
  - Allocates a voice from pool (may preempt lower-priority voice)
  - Registers `GUSWAVE_CallBack` or `GUSWAVE_DebugCallBack` with GUS hardware
  - Starts playback via `gf1_play_digital()`
- **Calls**: `GUSWAVE_AllocVoice()`, `GUSWAVE_GetNextVOCBlock()`, `GUSWAVE_Play()`
- **Notes**: Validates VOC signature; supports 8-bit mono only (stereo and packed VOC blocks are skipped). Returns error if no valid sound data found.

### GUSWAVE_PlayWAV

- **Signature**: `int GUSWAVE_PlayWAV(char *sample, int pitchoffset, int angle, int volume, int priority, unsigned long callbackval)`
- **Purpose**: Begin playback of a WAV-format sound file.
- **Inputs**: Same as `GUSWAVE_PlayVOC`
- **Outputs/Return**: Voice handle on success; error code on failure
- **Side effects**: Same as `GUSWAVE_PlayVOC`; allocates voice and starts hardware playback
- **Calls**: `GUSWAVE_AllocVoice()`, `GUSWAVE_Play()`
- **Notes**: Validates RIFF/WAVE headers. Supports 8-bit or 16-bit mono/stereo PCM (format tag 1). Loops not currently implemented (all loop pointers set to NULL).

### GUSWAVE_StartDemandFeedPlayback

- **Signature**: `int GUSWAVE_StartDemandFeedPlayback(void (*function)(char **ptr, unsigned long *length), int channels, int bits, int rate, int pitchoffset, int angle, int volume, int priority, unsigned long callbackval)`
- **Purpose**: Begin playback of streamed/demand-fed audio (callback-driven data source).
- **Inputs**: 
  - `function` – callback to fetch audio blocks: receives pointers to `*ptr` and `*length` (in/out)
  - `channels`, `bits`, `rate` – audio format specification
  - Other inputs same as `GUSWAVE_PlayVOC`
- **Outputs/Return**: Voice handle on success; error on failure
- **Side effects**: Allocates voice; initializes with silence buffer; registers callback
- **Calls**: `GUSWAVE_AllocVoice()`, `GUSWAVE_Play()`
- **Notes**: Client callback is responsible for providing fresh audio data on each ISR call. Useful for real-time synthesis or streaming decoders.

### GUSWAVE_Kill

- **Signature**: `int GUSWAVE_Kill(int handle)`
- **Purpose**: Stop playback of the voice with the specified handle.
- **Inputs**: `handle` – voice handle returned by a Play function
- **Outputs/Return**: `GUSWAVE_Ok` on success; `GUSWAVE_Warning` if voice not found
- **Side effects**: Calls `gf1_stop_digital()` to halt hardware playback; does not deallocate the voice (deallocation happens in ISR callback)
- **Calls**: `GUSWAVE_GetVoice()`, `gf1_stop_digital()`
- **Notes**: Disables interrupts during voice lookup; interrupt-safe.

### GUSWAVE_KillAllVoices

- **Signature**: `int GUSWAVE_KillAllVoices(void)`
- **Purpose**: Stop all active voices and reset state.
- **Inputs**: None
- **Outputs/Return**: `GUSWAVE_Ok`
- **Side effects**: 
  - Stops all active voices via `gf1_stop_digital()`
  - Clears `VoiceStatus` and linked lists
  - Resets all voice `Active` flags
  - Does not deallocate voice memory
- **Calls**: `gf1_stop_digital()`, `DisableInterrupts()`, `RestoreInterrupts()`, `LL_AddToTail()`
- **Notes**: Called during shutdown and when reallocating voices after priority preemption.

### GUSWAVE_SetPitch

- **Signature**: `int GUSWAVE_SetPitch(int handle, int pitchoffset)`
- **Purpose**: Change the playback pitch of an active voice.
- **Inputs**: 
  - `handle` – voice handle
  - `pitchoffset` – pitch shift in semitones (passed to `PITCH_GetScale()`)
- **Outputs/Return**: `GUSWAVE_Ok` on success; `GUSWAVE_Warning` if voice not found
- **Side effects**: Calls `gf1_dig_set_freq()` to update hardware playback rate
- **Calls**: `GUSWAVE_GetVoice()`, `PITCH_GetScale()`, `gf1_dig_set_freq()`, `DisableInterrupts()`, `RestoreInterrupts()`
- **Notes**: Interrupt-safe; recalculates `RateScale` from pitch and base sampling rate.

### GUSWAVE_SetPan3D

- **Signature**: `int GUSWAVE_SetPan3D(int handle, int angle, int distance)`
- **Purpose**: Set 3D pan and distance attenuation for a voice.
- **Inputs**: 
  - `handle` – voice handle
  - `angle` – direction (0–31, masked; 0=center, 16=opposite)
  - `distance` – distance attenuation (0–255; 0=closest, 255=farthest)
- **Outputs/Return**: `GUSWAVE_Ok` on success; `GUSWAVE_Warning` if voice not found
- **Side effects**: 
  - Looks up pan value in `GUSWAVE_PanTable`; applies `GUSWAVE_SwapLeftRight` inversion
  - Calls `gf1_dig_set_pan()` and `gf1_dig_set_vol()` to update hardware
- **Calls**: `GUSWAVE_GetVoice()`, `gf1_dig_set_pan()`, `gf1_dig_set_vol()`, `DisableInterrupts()`, `RestoreInterrupts()`
- **Notes**: Distance is converted to volume attenuation (volume = 255 - distance); interrupt-safe.

### GUSWAVE_SetVolume

- **Signature**: `void GUSWAVE_SetVolume(int volume)`
- **Purpose**: Set the master volume level for all voices.
- **Inputs**: `volume` – master volume (0–255; clamped)
- **Outputs/Return**: None
- **Side effects**: 
  - Updates `GUSWAVE_Volume` (internal 0–4095 range)
  - Updates hardware volume for all active voices via `gf1_dig_set_vol()`
- **Calls**: `gf1_dig_set_vol()` for each active voice
- **Notes**: Master volume is applied multiplicatively to per-voice volumes.

### GUSWAVE_VoicePlaying

- **Signature**: `int GUSWAVE_VoicePlaying(int handle)`
- **Purpose**: Check if a voice with the specified handle is currently active.
- **Inputs**: `handle` – voice handle
- **Outputs/Return**: `TRUE` if voice is active; `FALSE` otherwise
- **Side effects**: Sets error code if voice not found
- **Calls**: `GUSWAVE_GetVoice()`
- **Notes**: Simple wrapper around voice lookup and `Active` flag check.

### GUSWAVE_VoicesPlaying

- **Signature**: `int GUSWAVE_VoicesPlaying(void)`
- **Purpose**: Count the number of currently active voices.
- **Inputs**: None
- **Outputs/Return**: Count of active voices
- **Side effects**: None
- **Calls**: `DisableInterrupts()`, `RestoreInterrupts()`
- **Notes**: Interrupt-safe; used internally by allocation logic. May log via debug output.

### GUSWAVE_GetNextVOCBlock

- **Signature**: `playbackstatus GUSWAVE_GetNextVOCBlock(VoiceNode *voice)`
- **Purpose**: Parse the next block from a VOC file and update voice state.
- **Inputs**: `voice` – voice node with current playback position
- **Outputs/Return**: `KeepPlaying` if more data available; `SoundDone` if end of file
- **Side effects**: 
  - Advances `voice->NextBlock` pointer through VOC file
  - Updates `voice->sound`, `voice->length`, `voice->BlockLength`, `voice->SamplingRate`, `voice->bits`
  - Handles VOC repeat/loop blocks (blocks 6–7)
- **Calls**: None (state machine only)
- **Notes**: 
  - Implements VOC file format parser (blocks 0–9)
  - Skips unsupported formats: stereo, packed data, silence, markers, ASCII
  - Supports 8-bit block (1), continuation (2), extended (8), and new sound data (9)
  - Implements repeat loops (blocks 6–7) with count tracking

### GUSWAVE_GetNextWAVBlock

- **Signature**: `playbackstatus GUSWAVE_GetNextWAVBlock(VoiceNode *voice)`
- **Purpose**: Fetch the next audio block from a WAV file.
- **Inputs**: `voice` – voice node with current playback position
- **Outputs/Return**: `KeepPlaying` if more data available; `SoundDone` if loop end reached
- **Side effects**: 
  - Advances `voice->sound` and `voice->NextBlock` pointers
  - Updates `voice->length` and `voice->BlockLength`
  - Handles loop restart if `voice->LoopStart` is set
- **Calls**: None
- **Notes**: Simplified streaming; does not currently support loop points (all loop pointers are NULL in `GUSWAVE_PlayWAV`).

### GUSWAVE_GetNextDemandFeedBlock

- **Signature**: `playbackstatus GUSWAVE_GetNextDemandFeedBlock(VoiceNode *voice)`
- **Purpose**: Fetch the next audio block from a client demand-feed callback.
- **Inputs**: `voice` – voice node with demand-feed function pointer
- **Outputs/Return**: `KeepPlaying` or `NoMoreData` depending on callback result
- **Side effects**: 
  - Calls `voice->DemandFeed()` callback to fetch data
  - Updates `voice->sound`, `voice->length`, `voice->BlockLength`
- **Calls**: `voice->DemandFeed()` callback
- **Notes**: Client callback is responsible for managing audio buffer lifecycle.

### GUSWAVE_CallBack

- **Signature**: `static int LOADDS GUSWAVE_CallBack(int reason, int voice, unsigned char **buf, unsigned long *size)`
- **Purpose**: Interrupt service routine callback invoked by GUS hardware during/after playback.
- **Inputs**: 
  - `reason` – `DIG_MORE_DATA` (need next block) or `DIG_DONE` (playback finished)
  - `voice` – hardware voice number
  - `buf` – (output) pointer to next audio buffer
  - `size` – (output) size of buffer in bytes
- **Outputs/Return**: `DIG_DONE` or `DIG_MORE_DATA`
- **Side effects (global state, I/O, alloc)**: 
  - On `DIG_MORE_DATA`: calls `voice->GetSound()` to fetch next block; may set `*buf` to silence if no data
  - On `DIG_DONE`: marks voice as inactive; removes from `VoiceList`; adds to `VoicePool`; calls user callback
  - Modifies `VoiceStatus[voice].playing` and `VoiceStatus[voice].Voice`
- **Calls**: `voice->GetSound()`, `LL_Remove()`, `LL_AddToTail()`, user callback via `GUSWAVE_CallBackFunc()`
- **Notes**: 
  - Executes in interrupt context; must not make DOS/BIOS calls or call non-reentrant C library functions
  - Disables certain debugging macros in production build (SetBorderColor commented out)

### GUSWAVE_DebugCallBack

- **Signature**: Same as `GUSWAVE_CallBack`
- **Purpose**: Debug variant of callback with verbose logging.
- **Inputs/Outputs/Return**: Same as `GUSWAVE_CallBack`
- **Side effects**: Same as `GUSWAVE_CallBack`, plus debug output via `DB_printf()` and `DB_PrintNum()`
- **Calls**: `DB_printf()`, `DB_PrintNum()` for logging; otherwise same as `GUSWAVE_CallBack`
- **Notes**: Used when `GUS_Debug` flag is set; helps diagnose playback issues in interrupt context.

### GUSWAVE_AllocVoice

- **Signature**: `static VoiceNode *GUSWAVE_AllocVoice(int priority)`
- **Purpose**: Retrieve an inactive voice or preempt a lower-priority active voice.
- **Inputs**: `priority` – priority level of request
- **Outputs/Return**: Pointer to allocated `VoiceNode` on success; `NULL` if no voices available and no lower-priority voice to preempt
- **Side effects**: 
  - Removes voice from `VoicePool`
  - May call `GUSWAVE_Kill()` to preempt lower-priority voice
  - Assigns new unique handle via `GUSWAVE_VoiceHandle` counter
- **Calls**: `GUSWAVE_VoicesPlaying()`, `DisableInterrupts()`, `RestoreInterrupts()`, `GUSWAVE_Kill()`, `GUSWAVE_VoicePlaying()`
- **Notes**: 
  - Interrupt-safe; acquires flags around linked list operations
  - Scans `VoiceList` for lowest-priority voice if pool is empty

### GUSWAVE_GetVoice

- **Signature**: `static VoiceNode *GUSWAVE_GetVoice(int handle)`
- **Purpose**: Look up a voice by its handle.
- **Inputs**: `handle` – voice handle
- **Outputs/Return**: Pointer to `VoiceNode` on success; `NULL` if not found (also sets error code)
- **Side effects**: Sets `GUSWAVE_ErrorCode` if voice not found
- **Calls**: `DisableInterrupts()`, `RestoreInterrupts()`
- **Notes**: Interrupt-safe; linear search through `VoiceList`.

---

## Control Flow Notes

**Initialization phase**:
- `GUSWAVE_Init()` → `GUS_Init()` (hardware), then `GUSWAVE_InitVoices()` (allocate GUS memory, populate `VoicePool`)
- Voices start in `VoicePool` (inactive, not linked in `VoiceList`)

**Playback startup**:
1. Client calls `GUSWAVE_PlayVOC()`, `GUSWAVE_PlayWAV()`, or `GUSWAVE_StartDemandFeedPlayback()`
2. Function allocates voice via `GUSWAVE_AllocVoice()` (may preempt lower priority)
3. Audio format is parsed; voice state (sampling rate, bits, GetSound function pointer) is initialized
4. `GUSWAVE_Play()` is called, which invokes `gf1_play_digital()` to start hardware playback and register ISR callback
5. Voice moves from `VoicePool` to `VoiceList`

**During playback (interrupt context)**:
- GUS hardware repeatedly calls registered callback (`GUSWAVE_CallBack` or debug variant)
- On `DIG_MORE_DATA`: callback invokes `voice->GetSound()` (one of `GUSWAVE_GetNextVOCBlock()`, `GUSWAVE_GetNextWAVBlock()`, `GUSWAVE_GetNextDemandFeedBlock()`)
- If sound data available, callback returns pointer and size; hardware continues playback
- If `SoundDone` or error, callback fills buffer with silence and returns `DIG_DONE`

**Playback completion**:
- When hardware finishes (after all data consumed), callback receives `DIG_DONE` reason
- Callback marks voice as inactive, removes from `VoiceList`, adds back to `VoicePool`
- User callback is invoked with stored `callbackval`

**Voice control**:
- At any time, client can call `GUSWAVE_SetPitch()`, `GUSWAVE_SetPan3D()` to modify active voice
- `GUSWAVE_Kill()` stops a voice immediately (unlike natural completion, does not wait for data exhaustion)
- `GUSWAVE_KillAllVoices()` stops all and resets state (used during shutdown or re-initialization)

## External Dependencies

**Standard C / DOS headers**:
- `<stdlib.h>`, `<conio.h>`, `<dos.h>`, `<stdio.h>`, `<io.h>`, `<string.h>` – memory, I/O, DOS services

**Custom headers**:
- `debugio.h` – debug output (`DB_printf()`, `DB_PrintNum()`)
- `interrup.h` – interrupt control (`DisableInterrupts()`, `RestoreInterrupts()`)
- `ll_man.h` – linked list macros (`LL_AddToTail()`, `LL_Remove()`)
- `pitch.h` – pitch scaling (`PITCH_GetScale()`)
- `user.h` – command-line parameter checking (`USER_CheckParameter()`)
- `multivoc.h` – audio library API (error codes like `MV_InvalidVOCFile`)
- `_guswave.h` – private guswave declarations (likely `VoiceNode`, `voicestatus`, etc.)
- `newgf1.h` – GUS hardware interface (`gf1_play_digital()`, `gf1_stop_digital()`, etc.)
- `gusmidi.h` – MIDI subsystem detection (`GUSMIDI_Installed`)
- `guswave.h` – public header for this module

**External symbols (defined elsewhere)**:
- `GUS_Init()`, `GUS_Shutdown()`, `GUS_Error`, `GUS_ErrorString()` – GUS hardware initialization/control
- `gf1_play_digital()` – start hardware voice playback with callback
- `gf1_stop_digital()` – stop hardware voice
- `gf1_dig_set_freq()`, `gf1_dig_set_pan()`, `gf1_dig_set_vol()` – hardware voice parameter updates
- `gf1_malloc()` – allocate GUS onboard memory
- `GUS_HoldBuffer` – internal GUS buffer for playback handoff
- `GF1BSIZE` – GUS voice buffer size constant
- `GUSMIDI_Installed` – MIDI module presence flag
- `MV_ErrorCode` – multivoc error reporting
- `GUSWAVE_MinVoiceHandle`, `MAX_VOICES`, `VOICES`, `MAX_VOLUME`, `VOC_8BIT`, `VOC_16BIT`, `MAX_BLOCK_LENGTH` – constants (likely from headers)
