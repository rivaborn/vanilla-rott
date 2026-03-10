# audiolib/source/midi.h

## File Purpose
Public header for MIDI.C declaring the interface for MIDI song file playback and control. Defines error codes, callback function structure, and API for playing, pausing, stopping, and configuring MIDI music during gameplay.

## Core Responsibilities
- Define MIDI error codes and return status enumeration
- Declare callback function interface (`midifuncs` struct) for MIDI event routing to synthesis/output engine
- Provide playback control (play, pause, stop, continue, query playing status)
- Provide tempo and position control (set/get tempo, seek by ticks/time/measure-beat-tick)
- Provide volume control (global and per-channel)
- Provide MIDI system configuration (patch mapping, context, loop flags)
- Provide resource lifecycle management (lock/unlock memory, load timbres, reset)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MIDI_Errors` | enum | Error/status codes for MIDI operations |
| `midifuncs` | struct | Callback function pointers for MIDI synthesis and event handling |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MIDI_PatchMap` | `char[128]` | extern global | Instrument patch mapping table (128 MIDI programs) |
| `MIDI_PASS_THROUGH` | #define (1) | file | Flag constant for MIDI routing mode |
| `MIDI_DONT_PLAY` | #define (0) | file | Flag constant for non-playback mode |
| `MIDI_MaxVolume` | #define (255) | file | Maximum MIDI volume level |

## Key Functions / Methods

### MIDI_PlaySong
- Signature: `int MIDI_PlaySong(unsigned char *song, int loopflag)`
- Purpose: Start playback of a MIDI song from memory
- Inputs: `song` (pointer to MIDI data in memory), `loopflag` (loop mode)
- Outputs/Return: Status code (MIDI_Ok on success, error code on failure)
- Side effects: Initializes playback state, begins song rendering
- Notes: Main entry point for music playback

### MIDI_StopSong
- Signature: `void MIDI_StopSong(void)`
- Purpose: Immediately stop MIDI playback and silence all notes
- Outputs/Return: None
- Side effects: Stops playback state, calls ReleasePatches callback

### MIDI_PauseSong / MIDI_ContinueSong
- Signature: `void MIDI_PauseSong(void)` / `void MIDI_ContinueSong(void)`
- Purpose: Pause/resume playback without resetting song position
- Side effects: Suspends/resumes MIDI event processing

### MIDI_SongPlaying
- Signature: `int MIDI_SongPlaying(void)`
- Purpose: Query whether a song is currently playing
- Outputs/Return: Non-zero if playing, zero if stopped/paused

### MIDI_SetMidiFuncs
- Signature: `void MIDI_SetMidiFuncs(midifuncs *funcs)`
- Purpose: Register callback function pointers for MIDI event routing to synthesis engine
- Inputs: Struct containing function pointers for NoteOn, NoteOff, ControlChange, ProgramChange, PitchBend, volume control, etc.
- Side effects: Sets global callback dispatch table

### MIDI_SetVolume / MIDI_GetVolume
- Signature: `int MIDI_SetVolume(int volume)` / `int MIDI_GetVolume(void)`
- Purpose: Set/query global master MIDI volume (0–255)
- Inputs/Outputs: Volume level

### MIDI_SetTempo / MIDI_GetTempo
- Signature: `void MIDI_SetTempo(int tempo)` / `int MIDI_GetTempo(void)`
- Purpose: Adjust playback speed (tempo in BPM or ticks/minute)
- Side effects: Affects event timing during playback

### MIDI_SetSongPosition (position variants)
- Signature: `void MIDI_SetSongTick(unsigned long PositionInTicks)` / `void MIDI_SetSongTime(unsigned long milliseconds)` / `void MIDI_SetSongPosition(int measure, int beat, int tick)`
- Purpose: Seek to arbitrary position in song
- Inputs: Position in different units (ticks, milliseconds, or musical coordinates)
- Side effects: Resets playback head, may trigger note-off for in-flight notes

### MIDI_GetSongPosition / MIDI_GetSongLength
- Signature: `void MIDI_GetSongPosition(songposition *pos)` / `void MIDI_GetSongLength(songposition *pos)`
- Purpose: Query current playback position or total song length
- Outputs: Filled songposition struct (defined elsewhere)

### MIDI_RerouteMidiChannel
- Signature: `void MIDI_RerouteMidiChannel(int channel, int cdecl (*function)(int event, int c1, int c2))`
- Purpose: Redirect MIDI events on a channel to a custom handler function (for effects, routing, mixing)
- Inputs: MIDI channel (0–15), custom event callback
- Side effects: Modifies channel routing table

### MIDI_Reset
- Signature: `int MIDI_Reset(void)`
- Purpose: Reset MIDI system to clean state (stop all notes, reset controllers, release patches)
- Outputs/Return: Status code

**Trivial helpers summarized under Notes:**
- `MIDI_AllNotesOff`, `MIDI_SetUserChannelVolume`, `MIDI_ResetUserChannelVolume`: Per-channel and global state management
- `MIDI_SetContext`, `MIDI_GetContext`: Context/subsystem ID management
- `MIDI_SetLoopFlag`: Set loop behavior for current song
- `MIDI_LoadTimbres`, `MIDI_LockMemory`, `MIDI_UnlockMemory`: Resource lifecycle (patch loading, DOS DPMI memory locking)

## Control Flow Notes
This module is invoked during game init (MIDI_LoadTimbres, MIDI_LockMemory, MIDI_SetMidiFuncs) and shutdown (MIDI_UnlockMemory, MIDI_Reset). During main loop, MIDI_SongPlaying and playback position queries inform UI/HUD. Song playback initiated via MIDI_PlaySong typically early in game state transitions (menu music, level music).

## External Dependencies
- **Defined elsewhere**: `songposition` struct (type used in position/length queries)
- **Callback interface consumers**: Synthesis engine or hardware output driver that implements `midifuncs` callbacks (NoteOn, NoteOff, ControlChange, etc.)
