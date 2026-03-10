# audiolib/source/music.h

## File Purpose
Public API header for the MUSIC.C module. Declares functions and types for MIDI/music playback control, including song loading, playback state management, volume control, and position seeking in the Apogee Sound System (ASS).

## Core Responsibilities
- Initialize and shut down the music/MIDI subsystem
- Load and play MIDI song data with loop control
- Control playback state (play, pause, stop, continue)
- Manage master and per-channel MIDI volume
- Seek to specific song positions (ticks, milliseconds, measures/beats)
- Apply fade effects to volume over time
- Configure MIDI channel routing and timbre banks
- Report music system errors

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `songposition` | struct | Encodes playback position in multiple formats: absolute tick count, milliseconds, and measure/beat/tick notation |
| `MUSIC_ERRORS` | enum | Error codes returned by music functions (Warning, Error, Ok, plus various failure conditions like FMNotDetected, DPMI_Error) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MUSIC_ErrorCode` | int | global | Stores the most recent error code from music operations |

## Key Functions / Methods

### MUSIC_Init
- Signature: `int MUSIC_Init( int SoundCard, int Address )`
- Purpose: Initialize the music subsystem with specified sound card and hardware address
- Inputs: SoundCard (enum from sndcards.h), Address (hardware I/O address)
- Outputs/Return: Error code (MUSIC_ERRORS enum value)
- Side effects: Initializes global music state, configures hardware
- Notes: Must be called before any other MUSIC_* function

### MUSIC_Shutdown
- Signature: `int MUSIC_Shutdown( void )`
- Purpose: Gracefully shut down the music subsystem
- Outputs/Return: Error code
- Side effects: Releases hardware resources, stops playback

### MUSIC_PlaySong
- Signature: `int MUSIC_PlaySong( unsigned char *song, int loopflag )`
- Purpose: Load and start playback of a MIDI song
- Inputs: song (pointer to MIDI data), loopflag (MUSIC_LoopSong or MUSIC_PlayOnce)
- Outputs/Return: Error code
- Side effects: Initiates playback, updates MIDI_ErrorCode

### MUSIC_SetVolume / MUSIC_GetVolume
- Set/query master playback volume (0–100 scale implied)

### MUSIC_SetMidiChannelVolume / MUSIC_ResetMidiChannelVolumes
- Control per-channel MIDI volume or reset all channels to default

### MUSIC_FadeVolume
- Signature: `int MUSIC_FadeVolume( int tovolume, int milliseconds )`
- Purpose: Smoothly fade volume over specified duration
- Returns: Error code; check MUSIC_FadeActive() to monitor fade state

### MUSIC_SetSongTick / MUSIC_SetSongTime / MUSIC_SetSongPosition
- Seek to playback position by ticks, milliseconds, or measure/beat/tick notation

### MUSIC_GetSongPosition / MUSIC_GetSongLength
- Query current playback position and total song length (fills songposition struct)

### MUSIC_RerouteMidiChannel
- Signature: `void MUSIC_RerouteMidiChannel( int channel, int cdecl ( *function )( int event, int c1, int c2 ) )`
- Purpose: Redirect MIDI events from a channel to a custom callback instead of hardware
- Inputs: channel (MIDI channel), function pointer (receives MIDI event, controller1, controller2)
- Notes: Allows software synthesis or custom handling per MIDI channel

### MUSIC_RegisterTimbreBank
- Register a timbre/instrument bank for FM or wavetable synthesis

## Control Flow Notes
Typical lifecycle: **MUSIC_Init** → **MUSIC_PlaySong** → (playback control: pause/continue/seek) → **MUSIC_StopSong** → **MUSIC_Shutdown**. The music system runs asynchronously; playback continues until explicitly stopped or song ends (unless looped). Context switching (MUSIC_SetContext / MUSIC_GetContext) allows multiple music states.

## External Dependencies
- **Include**: `sndcards.h` (defines soundcardnames enum)
- **Defined elsewhere**: All function implementations in MUSIC.C; hardware abstraction and MIDI I/O handled by sound card drivers
