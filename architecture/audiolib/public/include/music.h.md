# audiolib/public/include/music.h

## File Purpose
Public header for the MUSIC.C module providing the interface for MIDI music playback and control in the game engine. Defines error codes, song timing structures, and function signatures for initializing, playing, controlling, and monitoring music playback across different sound cards and MIDI devices.

## Core Responsibilities
- Initialize/shutdown music system with specified sound card and I/O address
- Play, pause, continue, and stop MIDI songs with loop control
- Control global and per-MIDI-channel volume, including fade effects
- Seek/position songs by ticks, milliseconds, or measure/beat/tick notation
- Query song playback state (playing, position, length)
- Configure MIDI channel mapping and timbre banks
- Route MIDI channels to custom callback handlers
- Report errors via error codes and human-readable messages

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MUSIC_ERRORS` | enum | Error codes for music operations (Ok, ASSVersion, SoundCardError, MPU401Error, etc.) |
| `songposition` | struct | Song timing: tickposition, milliseconds, measure, beat, tick |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MUSIC_ErrorCode` | int | global | Stores error code from last music operation |

## Key Functions / Methods

### MUSIC_Init
- Signature: `int MUSIC_Init(int SoundCard, int Address)`
- Purpose: Initialize music system with specified sound card and I/O base address
- Inputs: SoundCard (card type from sndcards.h), Address (I/O port)
- Outputs/Return: Error code (MUSIC_Ok or error enum value)
- Side effects: Allocates/initializes MIDI driver resources, sets global error code
- Calls: (defined elsewhere—likely calls sound card driver initialization)
- Notes: Must be called before any music operations; pairs with MUSIC_Shutdown()

### MUSIC_Shutdown
- Signature: `int MUSIC_Shutdown(void)`
- Purpose: Clean up and disable music system
- Inputs: None
- Outputs/Return: Error code
- Side effects: Releases MIDI resources, stops any playing song
- Calls: (defined elsewhere)
- Notes: Should be called at game shutdown

### MUSIC_PlaySong
- Signature: `int MUSIC_PlaySong(unsigned char *song, int loopflag)`
- Purpose: Load and play a MIDI song
- Inputs: song (pointer to MIDI data), loopflag (MUSIC_LoopSong or MUSIC_PlayOnce)
- Outputs/Return: Error code
- Side effects: Stops current song, begins playback, sets loop mode
- Calls: (defined elsewhere)
- Notes: Song data format inferred to be raw MIDI; loopflag controls whether song restarts after completion

### MUSIC_SetVolume / MUSIC_SetMidiChannelVolume
- Signature: `void MUSIC_SetVolume(int volume)` / `void MUSIC_SetMidiChannelVolume(int channel, int volume)`
- Purpose: Set global or per-channel MIDI volume
- Inputs: volume (0–max, likely 0–127 for MIDI), channel (0–15 for MIDI channels)
- Outputs/Return: None
- Side effects: Updates sound output levels, applies to currently playing song
- Calls: (defined elsewhere)
- Notes: Affects playback immediately; per-channel allows independent instrument control

### MUSIC_SetSongTick / MUSIC_SetSongTime / MUSIC_SetSongPosition
- Signature: `void MUSIC_SetSongTick(unsigned long PositionInTicks)` / `void MUSIC_SetSongTime(unsigned long milliseconds)` / `void MUSIC_SetSongPosition(int measure, int beat, int tick)`
- Purpose: Seek song to specified position using different time units
- Inputs: Position (ticks, milliseconds, or measure/beat/tick tuple)
- Outputs/Return: None
- Side effects: Repositions playback; may cause brief audio discontinuity
- Calls: (defined elsewhere)
- Notes: Supports three seeking modes; internal conversion between units inferred

### MUSIC_FadeVolume
- Signature: `int MUSIC_FadeVolume(int tovolume, int milliseconds)`
- Purpose: Smoothly fade volume to target over specified duration
- Inputs: tovolume (target volume), milliseconds (fade duration)
- Outputs/Return: Error code
- Side effects: Initiates gradual volume change; affects playback until fade completes
- Calls: (defined elsewhere)
- Notes: Non-blocking; use MUSIC_FadeActive() to poll completion

### MUSIC_RerouteMidiChannel
- Signature: `void MUSIC_RerouteMidiChannel(int channel, int cdecl (*function)(int event, int c1, int c2))`
- Purpose: Redirect a MIDI channel to a custom callback function
- Inputs: channel (MIDI channel 0–15), function pointer (cdecl convention, event + 2 params)
- Outputs/Return: None
- Side effects: Intercepts MIDI events on specified channel, routes to custom handler
- Calls: (defined elsewhere; invokes callback)
- Notes: Allows custom MIDI processing (e.g., percussion synthesis, pitch shifting)

### MUSIC_RegisterTimbreBank
- Signature: `void MUSIC_RegisterTimbreBank(unsigned char *timbres)`
- Purpose: Load custom FM/MIDI instrument definitions
- Inputs: timbres (pointer to timbre/instrument data)
- Outputs/Return: None
- Side effects: Replaces instrument set; affects all subsequent song playbacks
- Calls: (defined elsewhere)
- Notes: Supports SoundBlaster FM synthesis and MIDI devices

### Utility Functions (1–2 bullets under Notes)
- `MUSIC_Pause`, `MUSIC_Continue`, `MUSIC_StopSong`: Song control (play state)
- `MUSIC_SongPlaying`, `MUSIC_GetSongPosition`, `MUSIC_GetSongLength`: Query playback state
- `MUSIC_SetLoopFlag`, `MUSIC_SetContext`, `MUSIC_GetContext`, `MUSIC_ResetMidiChannelVolumes`: Configuration helpers
- `MUSIC_ErrorString`: Return human-readable error message for error code
- `MUSIC_GetVolume`: Query current volume
- `MUSIC_SetMaxFMMidiChannel`, `MUSIC_FadeActive`, `MUSIC_StopFade`: Advanced control

## Control Flow Notes
**Initialization → Playback → Shutdown**
1. **Init phase**: Call `MUSIC_Init(soundcard, address)` to initialize driver
2. **Configuration phase**: `MUSIC_SetVolume()`, `MUSIC_RegisterTimbreBank()`, `MUSIC_SetMidiChannelVolume()` to prepare
3. **Playback phase**: `MUSIC_PlaySong()` starts music; `MUSIC_Pause()` / `MUSIC_Continue()` for pause/resume; `MUSIC_SetSongTime()` etc. for seeking
4. **Cleanup phase**: `MUSIC_Shutdown()` on engine shutdown

This module is typically called from game's main audio manager during frame updates (for fade timing) and in response to user/script events (play, stop, seek).

## External Dependencies
- **sndcards.h**: Sound card type definitions (`soundcardnames` enum)
- **Defined elsewhere**: All function implementations (MUSIC.C), MIDI driver layer, sound card initialization
