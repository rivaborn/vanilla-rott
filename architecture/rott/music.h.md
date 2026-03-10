# rott/music.h

## File Purpose
Public API header for the MUSIC subsystem. Declares functions for initializing MIDI/music devices, playback control, volume management, song positioning, and real-time MIDI channel manipulation in the ROTT game engine.

## Core Responsibilities
- Initialize and shutdown music/MIDI hardware
- Control music playback state (play, pause, continue, stop)
- Manage global and per-channel volume levels
- Support song position seeking (by ticks, milliseconds, or measure/beat/tick)
- Provide fade-out effects and loop control
- Support MIDI channel rerouting to custom handlers
- Register timbre/instrument banks
- Expose error codes and context switching

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MUSIC_ERRORS` | enum | Error and warning codes returned by music system functions |
| `songposition` | struct | Position metadata: tick position, milliseconds, measure, beat, tick components |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MUSIC_ErrorCode` | int | global | Stores the most recent error code from music operations |

## Key Functions / Methods

### MUSIC_Init
- Signature: `int MUSIC_Init(int SoundCard, int Address)`
- Purpose: Initialize the music subsystem with a specified sound card device and hardware address.
- Inputs: `SoundCard` (device type from soundcardnames enum), `Address` (I/O port address)
- Outputs/Return: Error code (MUSIC_ERRORS enum value)
- Side effects: Initializes global MIDI/music hardware state, sets `MUSIC_ErrorCode`
- Calls: (Not visible in header)
- Notes: Must be called before any other music operations.

### MUSIC_Shutdown
- Signature: `int MUSIC_Shutdown(void)`
- Purpose: Deinitialize and release music hardware resources.
- Inputs: None
- Outputs/Return: Error code
- Side effects: Releases hardware resources, stops playback
- Calls: (Not visible in header)
- Notes: Inverse of MUSIC_Init.

### MUSIC_PlaySong
- Signature: `int MUSIC_PlaySong(unsigned char *song, int loopflag)`
- Purpose: Load and begin playback of a MIDI song.
- Inputs: `song` (raw MIDI data), `loopflag` (MUSIC_LoopSong or MUSIC_PlayOnce)
- Outputs/Return: Error code
- Side effects: Starts audio playback, updates internal playback state
- Calls: (Not visible in header)
- Notes: Song data format is raw MIDI.

### MUSIC_StopSong
- Signature: `int MUSIC_StopSong(void)`
- Purpose: Stop playback of the currently playing song.
- Outputs/Return: Error code
- Calls: (Not visible in header)

### MUSIC_Pause / MUSIC_Continue
- Signature: `void MUSIC_Pause(void)`, `void MUSIC_Continue(void)`
- Purpose: Pause and resume song playback without stopping.

### MUSIC_SetVolume / MUSIC_GetVolume
- Signature: `void MUSIC_SetVolume(int volume)`, `int MUSIC_GetVolume(void)`
- Purpose: Set/get overall music volume level.
- Inputs: `volume` (likely 0–255 or similar scale)

### MUSIC_SetSongPosition / MUSIC_GetSongPosition / MUSIC_GetSongLength
- Signature: `void MUSIC_SetSongPosition(int measure, int beat, int tick)`, `void MUSIC_GetSongPosition(songposition *pos)`, `void MUSIC_GetSongLength(songposition *pos)`
- Purpose: Seek to or retrieve current/total song position.
- Inputs: Measure, beat, tick triplet; or pointer to songposition struct
- Notes: Supports both absolute positioning and query.

### MUSIC_FadeVolume
- Signature: `int MUSIC_FadeVolume(int tovolume, int milliseconds)`
- Purpose: Smoothly transition volume over a specified duration.
- Inputs: Target volume level, fade duration in milliseconds
- Outputs/Return: Error code
- Side effects: Schedules volume interpolation
- Notes: Can be checked with MUSIC_FadeActive(); cancelled with MUSIC_StopFade().

### MUSIC_RerouteMidiChannel
- Signature: `void MUSIC_RerouteMidiChannel(int channel, int cdecl (*function)(int event, int c1, int c2))`
- Purpose: Redirect MIDI messages from a channel to a custom callback function.
- Inputs: MIDI channel number, callback function pointer (event, controller1, controller2)
- Side effects: Modifies MIDI routing at runtime
- Notes: Callback uses `cdecl` calling convention.

### MUSIC_RegisterTimbreBank
- Signature: `void MUSIC_RegisterTimbreBank(unsigned char *timbres)`
- Purpose: Register a custom timbre/instrument bank for FM or MIDI playback.
- Inputs: Raw timbre data buffer

## Control Flow Notes
This is a subsystem initialization and runtime management header. Control flow:
1. **Init phase**: `MUSIC_Init()` called during engine startup
2. **Runtime**: Songs loaded with `MUSIC_PlaySong()`, controlled via pause/continue/stop
3. **Per-frame**: Volume/position updates applied (fade, seeking)
4. **Shutdown**: `MUSIC_Shutdown()` during engine teardown

Not inferable whether there is a dedicated update/tick function; fade and MIDI processing may be interrupt-driven or polled externally.

## External Dependencies
- `#include "sndcards.h"` — Provides `soundcardnames` enum for sound card types
- MIDI/FM synthesis hardware interface (implementation in MUSIC.C, not visible here)
- DOS/protected-mode conventions (cdecl callbacks, hardware I/O addresses)
