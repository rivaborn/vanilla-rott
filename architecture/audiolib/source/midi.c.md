# audiolib/source/midi.c

## File Purpose
Core MIDI song playback engine. Loads MIDI files, parses and interprets MIDI events (note on/off, control changes, tempo/time-signature meta events), manages playback state (play/pause/stop/seek), implements volume control, and supports EMIDI extensions (track inclusion/exclusion, looping contexts).

## Core Responsibilities
- Parse MIDI file format (header validation, track enumeration, event data streams)
- Schedule and execute tick-based event interpretation via task manager
- Route MIDI commands to sound device via callback function table (midifuncs)
- Maintain global playback state: active/loaded flags, position counters (ticks, beats, measures, time)
- Manage per-channel and master volume with user-level overrides
- Support EMIDI features: context switching, loop points, track filtering, dynamic program/volume changes
- Provide seek operations (by tick, time, or measure/beat/tick)
- Lock/unlock memory for DOS real-mode interrupt safety

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| track | struct (extern) | Per-track playback state: current position, event delay, active flag, EMIDI feature flags, running status, context stack |
| midifuncs | struct (extern) | Callback function pointers for device output (NoteOn, NoteOff, ControlChange, ProgramChange, etc.) |
| songposition | struct (extern) | Song position in multiple representations: tick count, measure/beat/tick, milliseconds |
| task | struct (task_man.h) | Task scheduler entry for periodic _MIDI_ServiceRoutine invocation |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _MIDI_CommandLengths | const int[16] | static | Parameter byte count per MIDI command type (lookup table) |
| _MIDI_RerouteFunctions | function ptr[16] | static | Per-channel intercept callbacks for rerouting/filtering MIDI events |
| _MIDI_TrackPtr | track* | static | Allocated track array for current song |
| _MIDI_NumTracks | int | static | Number of tracks in loaded song |
| _MIDI_TrackMemSize | int | static | Byte size of track allocation |
| _MIDI_SongActive | int | static | Playback currently running (not paused) |
| _MIDI_SongLoaded | int | static | Song data resident in memory |
| _MIDI_Loop | int | static | Restart from beginning when song ends |
| _MIDI_PlayRoutine | task* | static | Task manager handle for service routine |
| _MIDI_Division | int | static | MIDI header: ticks per quarter note |
| _MIDI_Tick, _MIDI_Beat, _MIDI_Measure | int | static | Current playback position (sub-beat, beat, measure) |
| _MIDI_Time | unsigned | static | Elapsed time in fixed-point seconds |
| _MIDI_BeatsPerMeasure, _MIDI_TicksPerBeat, _MIDI_TimeBase | int | static | Time signature state |
| _MIDI_FPSecondsPerTick | long | static | Fixed-point conversion factor for tick-to-time |
| _MIDI_TotalTime, _MIDI_TotalTicks, _MIDI_TotalBeats, _MIDI_TotalMeasures | int/unsigned | static | Song length totals (measured during _MIDI_InitEMIDI) |
| _MIDI_PositionInTicks | unsigned long | static | Absolute tick counter |
| _MIDI_Context | int | static | Current EMIDI context index (-1 = unset) |
| _MIDI_ActiveTracks | int | static | Count of non-idle tracks |
| _MIDI_TotalVolume | int | static | Master volume (0–MIDI_MaxVolume) |
| _MIDI_ChannelVolume | int[16] | static | Per-channel volume set by control changes |
| _MIDI_UserChannelVolume | int[16] | static | Per-channel user multiplier (256 = 1.0) |
| _MIDI_Funcs | midifuncs* | static | Device callback table (set by MIDI_SetMidiFuncs) |
| Reset | int | static | Flag to reset device on next MIDI_PlaySong |
| MIDI_Tempo | int | global | Current tempo in BPM |
| MIDI_PatchMap | char[128] | global | Patch number remap table (for patch substitution) |

## Key Functions / Methods

### MIDI_PlaySong
- **Signature**: `int MIDI_PlaySong(unsigned char *song, int loopflag)`
- **Purpose**: Begin MIDI playback by validating file, allocating tracks, scanning EMIDI metadata, and scheduling service routine
- **Inputs**: `song` (pointer to MIDI file in memory), `loopflag` (loop on end)
- **Outputs/Return**: `MIDI_Ok`, `MIDI_NullMidiModule`, `MIDI_InvalidMidiFile`, `MIDI_UnknownMidiFormat`, `MIDI_NoTracks`, `MIDI_NoMemory`, `MIDI_Error`, `MIDI_InvalidTrack`
- **Side effects**: Allocates and locks track buffer (DPMI), calls MIDI_Reset, schedules task, sets song loaded/active state
- **Calls**: MIDI_StopSong, _MIDI_ReadNumber, USRHOOKS_GetMem, DPMI_LockMemory, _MIDI_InitEMIDI, MIDI_LoadTimbres, MIDI_Reset, _MIDI_ResetTracks, TS_ScheduleTask, MIDI_SetTempo, TS_Dispatch
- **Notes**: Validates header signature (MThd), format version, track count; handles SMPTE time divisions; deallocates on error; ~100 Hz task rate hardcoded

### _MIDI_ServiceRoutine
- **Signature**: `static void _MIDI_ServiceRoutine(task *Task)`
- **Purpose**: Task-scheduled callback (~100 Hz) that interprets and executes all pending MIDI events for current tick
- **Inputs**: `Task` (task handle, unused)
- **Outputs/Return**: void
- **Side effects**: Advances track positions, executes MIDI event callbacks, updates global timing counters, may reset/loop song
- **Calls**: GET_NEXT_EVENT, _MIDI_ReadDelta, _MIDI_SysEx, _MIDI_MetaEvent, _MIDI_InterpretControllerInfo, _MIDI_RerouteFunctions, _MIDI_Funcs callbacks, _MIDI_AdvanceTick, _MIDI_ResetTracks
- **Notes**: Core playback loop; early return if not active; processes all active tracks with zero delay per tick; called at interrupt level

### MIDI_StopSong
- **Signature**: `void MIDI_StopSong(void)`
- **Purpose**: Terminate playback and clean up resources
- **Inputs**: none
- **Outputs/Return**: void
- **Side effects**: Terminates task routine, calls MIDI_Reset, deallocates track buffer, calls ReleasePatches callback, resets all state
- **Calls**: TS_Terminate, MIDI_Reset, _MIDI_ResetTracks, DPMI_UnlockMemory, USRHOOKS_FreeMem
- **Notes**: Safe to call if song not loaded; no-op in that case

### MIDI_SetTempo
- **Signature**: `void MIDI_SetTempo(int tempo)`
- **Purpose**: Change playback speed and recalculate tick timing
- **Inputs**: `tempo` (BPM, e.g., 120)
- **Outputs/Return**: void
- **Side effects**: Updates MIDI_Tempo global, adjusts task routine frequency via TS_SetTaskRate, recalculates _MIDI_FPSecondsPerTick
- **Calls**: TS_SetTaskRate
- **Notes**: Called by MIDI_MetaEvent for embedded tempo changes and user code; formula: tickspersecond = (tempo × Division) / 60

### _MIDI_MetaEvent
- **Signature**: `static void _MIDI_MetaEvent(track *Track)`
- **Purpose**: Interpret MIDI meta event (end-of-track, tempo, time signature)
- **Inputs**: `Track` (current track being interpreted)
- **Outputs/Return**: void
- **Side effects**: Deactivates track on end-of-track; calls MIDI_SetTempo on tempo change; updates time signature and tick calculations
- **Calls**: GET_NEXT_EVENT, _MIDI_ReadNumber, MIDI_SetTempo
- **Notes**: Handles FF 2F (end of track), FF 51 (tempo), FF 58 (time signature); advances track position by event length

### _MIDI_InterpretControllerInfo
- **Signature**: `static int _MIDI_InterpretControllerInfo(track *Track, int TimeSet, int channel, int c1, int c2)`
- **Purpose**: Process control change events including EMIDI extensions
- **Inputs**: Track, TimeSet flag (whether timing was already set in this tick), channel, controller number, controller value
- **Outputs/Return**: Updated TimeSet flag
- **Side effects**: Updates channel volume, manages loop points and contexts, calls device callbacks
- **Calls**: _MIDI_SetChannelVolume, _MIDI_Funcs->ControlChange, _MIDI_Funcs->ProgramChange
- **Notes**: Handles 40+ MIDI and EMIDI controller types; context switching and looping involve saving/restoring full playback state; supports EMIDI_INFINITE loop count

### _MIDI_InitEMIDI
- **Signature**: `static void _MIDI_InitEMIDI(void)`
- **Purpose**: Scan all tracks to detect EMIDI features and measure total song length
- **Inputs**: none
- **Outputs/Return**: void
- **Side effects**: Sets EMIDI_* flags on tracks, populates context stack data, computes _MIDI_Total* (time, ticks, beats, measures)
- **Calls**: _MIDI_ResetTracks, _MIDI_AdvanceTick, GET_NEXT_EVENT, _MIDI_SysEx, _MIDI_MetaEvent, EMIDI_AffectsCurrentCard
- **Notes**: Runs once per MIDI_PlaySong; device type (MUSIC_SoundDevice) determines EMIDI feature set enabled; tracks scanned independently to completion

### MIDI_LoadTimbres
- **Signature**: `void MIDI_LoadTimbres(void)`
- **Purpose**: Pre-cache instrument patches for devices supporting patch loading
- **Inputs**: none
- **Outputs/Return**: void
- **Side effects**: Calls _MIDI_Funcs->LoadPatch for each unique program/rhythm patch
- **Calls**: GET_NEXT_EVENT, _MIDI_ReadDelta, _MIDI_Funcs->LoadPatch
- **Notes**: Iterates all tracks; finds MIDI_PROGRAM_CHANGE (channels 0–15) and MIDI_NOTE_ON (channel 9 = rhythm); called only if LoadPatch callback exists

### MIDI_SetVolume / MIDI_GetVolume
- **Signature**: `int MIDI_SetVolume(int volume)` / `int MIDI_GetVolume(void)`
- **Purpose**: Set/get master music volume
- **Inputs**: `volume` (0 to MIDI_MaxVolume)
- **Outputs/Return**: `MIDI_Ok` or `MIDI_NullMidiModule`; GetVolume returns actual volume
- **Side effects**: Clamps volume, updates _MIDI_TotalVolume, propagates to all channels
- **Calls**: _MIDI_SetChannelVolume, _MIDI_SendChannelVolumes
- **Notes**: If device has SetVolume callback, rerouted channels handled specially; otherwise per-channel recalculation; volume clamped [0, MIDI_MaxVolume]

### MIDI_SetSongTick / MIDI_SetSongTime / MIDI_SetSongPosition
- **Signature**: `void MIDI_SetSongTick(unsigned long PositionInTicks)` / `void MIDI_SetSongTime(unsigned long milliseconds)` / `void MIDI_SetSongPosition(int measure, int beat, int tick)`
- **Purpose**: Seek to absolute position (by tick, time, or measure/beat/tick)
- **Inputs**: Target position in requested format
- **Outputs/Return**: void
- **Side effects**: Pauses, resets if seeking backward, advances via _MIDI_ProcessNextTick until target, restores volume, resumes
- **Calls**: MIDI_PauseSong, _MIDI_ResetTracks, MIDI_Reset, _MIDI_ProcessNextTick, MIDI_SetVolume, MIDI_ContinueSong
- **Notes**: Inefficient (reprocesses events forward); backward seeks reset device; respects looping behavior

### _MIDI_ReadNumber
- **Signature**: `static long _MIDI_ReadNumber(void *from, size_t size)`
- **Purpose**: Parse big-endian multi-byte integer (1–4 bytes) from MIDI file
- **Inputs**: Pointer, byte count (capped at 4)
- **Outputs/Return**: Decoded unsigned 32-bit value
- **Side effects**: none
- **Calls**: none
- **Notes**: Used for MIDI header fields and meta event data

### _MIDI_ReadDelta
- **Signature**: `static long _MIDI_ReadDelta(track *ptr)`
- **Purpose**: Decode MIDI variable-length quantity (VLQ) encoding for delta times
- **Inputs**: Track with pos at encoded data
- **Outputs/Return**: Decoded delta time value
- **Side effects**: Advances track->pos past consumed bytes
- **Calls**: GET_NEXT_EVENT
- **Notes**: Implements MIDI VLQ: high bit is continuation flag, low 7 bits contribute to value

### MIDI_AllNotesOff
- **Signature**: `int MIDI_AllNotesOff(void)`
- **Purpose**: Send all-notes-off and pedal-off messages on all 16 channels
- **Inputs**: none
- **Outputs/Return**: `MIDI_Ok`
- **Side effects**: Sends three control changes per channel (0x40, 0x7B, 0x78)
- **Calls**: _MIDI_SendControlChange
- **Notes**: Used to silence hanging notes during pause/stop/seek

### MIDI_Reset
- **Signature**: `int MIDI_Reset(void)`
- **Purpose**: Initialize MIDI device to General MIDI defaults
- **Inputs**: none
- **Outputs/Return**: `MIDI_Ok`
- **Side effects**: Calls MIDI_AllNotesOff, disables interrupts, delays ~40 ms (CLOCKS_PER_SEC/24), sends RPN and data-entry controller changes, sets default volumes
- **Calls**: MIDI_AllNotesOff, DisableInterrupts, RestoreInterrupts, _MIDI_SendControlChange, _MIDI_SendChannelVolumes
- **Notes**: Sets pitch bend sensitivity to ±2 semitones via RPN; includes interrupt disable/restore for timing reliability

### MIDI_GetSongPosition / MIDI_GetSongLength
- **Signature**: `void MIDI_GetSongPosition(songposition *pos)` / `void MIDI_GetSongLength(songposition *pos)`
- **Purpose**: Query current/total song position in multiple formats
- **Inputs**: Pointer to songposition struct (filled out)
- **Outputs/Return**: void
- **Side effects**: Computes milliseconds from fixed-point time; populates tickposition, measure, beat, tick fields
- **Calls**: none
- **Notes**: GetSongLength fills position with total; GetSongPosition returns current

### Utility Functions (Brief)
- **MIDI_PauseSong / MIDI_ContinueSong**: Set _MIDI_SongActive to FALSE/TRUE
- **MIDI_SongPlaying**: Return _MIDI_SongActive
- **MIDI_SetMidiFuncs**: Assign callback table
- **MIDI_SetLoopFlag**: Set _MIDI_Loop
- **MIDI_SetContext / MIDI_GetContext**: EMIDI context switching
- **MIDI_SetUserChannelVolume / MIDI_ResetUserChannelVolume**: User-level per-channel volume
- **_MIDI_SetChannelVolume**: Apply volume multipliers and call device
- **_MIDI_SendChannelVolumes**: Propagate all channel volumes to device
- **_MIDI_AdvanceTick**: Increment tick counters with beat/measure rollover
- **_MIDI_ResetTracks**: Reset track positions and state to song start
- **_MIDI_SysEx**: Skip SysEx event data
- **_MIDI_ProcessNextTick**: Single-tick event interpreter (used by seek functions)
- **MIDI_LockMemory / MIDI_UnlockMemory**: DPMI lock/unlock for real-mode interrupt safety

## Control Flow Notes

**Initialization → Playback**:
1. MIDI_PlaySong validates file, allocates/locks tracks, calls _MIDI_InitEMIDI to scan features
2. MIDI_LoadTimbres (if device supports) pre-caches patches
3. MIDI_Reset initializes device
4. TS_ScheduleTask schedules _MIDI_ServiceRoutine at ~100 Hz
5. _MIDI_ServiceRoutine executes repeatedly: reads events from all active tracks, dispatches callbacks, advances global timing

**Event Processing per Tick** (_MIDI_ServiceRoutine):
- For each track with delay==0: read MIDI event, handle meta/sysex, apply running status, check reroute callbacks, dispatch device callbacks, fetch next delta
- Decrement all track delays
- If all tracks inactive: reset and loop (if enabled) or stop

**Seeking**: Pause → Reset if backward → Process ticks in a loop → Resume

**Shutdown** (MIDI_StopSong): Terminate task → Reset device → Free memory → Clear state

## External Dependencies
- **stdlib.h, time.h, dos.h, string.h**: Standard C
- **sndcards.h**: Sound card enumeration (SoundBlaster, GenMidi, Awe32, UltraSound, etc.)
- **interrup.h**: Interrupt disable/restore (inline asm)
- **dpmi.h**: DOS Protected Mode Interface (memory locking)
- **standard.h**: Type definitions
- **task_man.h**: Task scheduler (TS_ScheduleTask, TS_Terminate, TS_SetTaskRate, TS_Dispatch)
- **ll_man.h**: Linked list (included, use not visible)
- **usrhooks.h**: Memory allocation (USRHOOKS_GetMem, USRHOOKS_FreeMem)
- **music.h**: MIDI error codes and constants (defined elsewhere)
- **_midi.h**: Internal definitions—likely macros (GET_NEXT_EVENT, GET_MIDI_COMMAND, GET_MIDI_CHANNEL), track struct, MIDI event constants, EMIDI codes
- **midi.h**: Public API (user-facing declarations)
- **debugio.h**: Debug output (included but unused)

**Defined Elsewhere**:
- MUSIC_SoundDevice (global sound card type)
- MIDI_* error/status constants
- MIDI event command type constants (MIDI_NOTE_ON, MIDI_CONTROL_CHANGE, etc.)
- Meta event type constants (MIDI_TEMPO_CHANGE, MIDI_END_OF_TRACK, MIDI_TIME_SIGNATURE)
- EMIDI_* extension constants (EMIDI_LOOP_START, EMIDI_CONTEXT_START, etc.)
- EMIDI_AffectsCurrentCard(c2, type) function
- EMIDI_NUM_CONTEXTS, EMIDI_INFINITE, EMIDI_END_LOOP_VALUE
- MIDI_MaxVolume, GENMIDI_DefaultVolume
- NUM_MIDI_CHANNELS (16)
- MAX_FORMAT (max MIDI format version)
- TIME_PRECISION (fixed-point shift)
- RELATIVE_BEAT(measure, beat, tick) macro
