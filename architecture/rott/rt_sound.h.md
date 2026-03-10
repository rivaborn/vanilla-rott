# rott/rt_sound.h

## File Purpose

Public sound and music subsystem header for Rise of the Triad. Defines enums for all in-game sound effects and music, supported audio hardware types, and declares the API for sound playback, spatial audio, music management, and device initialization/shutdown.

## Core Responsibilities

- Define comprehensive enumeration of game sounds and music events
- Declare sound effect playback API (direct play, positioned, 3D, pitched)
- Declare music playback and fade control API
- Define supported audio card types (Adlib, General MIDI, Sound Blaster, etc.)
- Declare sound device setup, startup, and shutdown routines
- Provide macro wrappers for music volume/fade operations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `game_sounds` | enum | 300+ sound effect IDs covering menus, player, weapons, actors, environment, secrets |
| `remotesounds` | enum | 10 remote control sound IDs (shift+number row mapping) |
| `musesounds` | enum | 86 categorized music/MUSE format sound IDs (submix of game_sounds) |
| `fxtypes` | enum | Effect type selector: `fx_digital`, `fx_muse` |
| `ASSTypes` | enum | Audio hardware: UltraSound, SoundBlaster, AWE32, Adlib, GeneralMidi, TandySoundSource, PCSpeaker, Off |
| `songtypes` | enum | 18 music compositions (title, level, boss, cinematic, etc.) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SD_Started` | int (extern) | global | Flag indicating sound subsystem initialization status |

## Key Functions / Methods

### SD_SetupFXCard
- Signature: `int SD_SetupFXCard(int *numvoices, int *numbits, int *numchannels)`
- Purpose: Configure audio card parameters (voices, bit depth, channels)
- Inputs: Pointers to voice count, bit depth, channel count
- Outputs/Return: Status code
- Side effects: Modifies audio hardware state; populates output parameters
- Calls: (audio hardware driver, not visible)
- Notes: Called during initialization to probe/configure sound card

### SD_Startup
- Signature: `int SD_Startup(boolean bombonerror)`
- Purpose: Initialize sound subsystem
- Inputs: `bombonerror` – whether to abort on error
- Outputs/Return: Status code
- Side effects: Sets `SD_Started`, allocates audio resources
- Calls: (audio driver initialization, not visible)
- Notes: Sets up sound card, memory, interrupt handlers

### SD_Shutdown
- Signature: `void SD_Shutdown(void)`
- Purpose: Shut down sound subsystem and release resources
- Inputs: None
- Outputs/Return: None
- Side effects: Stops all sounds, deallocates memory, clears `SD_Started`
- Calls: (audio driver shutdown, not visible)
- Notes: Called at game exit

### SD_Play
- Signature: `int SD_Play(int sndnum)`
- Purpose: Play a sound effect immediately
- Inputs: Sound effect ID from `game_sounds` enum
- Outputs/Return: Handle for sound instance (for stopping/tracking)
- Side effects: Allocates voice; produces audio output
- Calls: Audio playback driver
- Notes: Non-spatial; uses default volume and positioning

### SD_PlayPositionedSound
- Signature: `int SD_PlayPositionedSound(int sndnum, int px, int py, int x, int y)`
- Purpose: Play sound with positional audio (panner coordinates and listener position)
- Inputs: Sound ID; panner point (px, py); listener position (x, y)
- Outputs/Return: Sound handle
- Side effects: Allocates voice; applies pan/volume based on relative position
- Calls: Audio driver with pan parameters
- Notes: Provides illusion of sound source in 2D space

### SD_PlaySoundRTP
- Signature: `int SD_PlaySoundRTP(int sndnum, int x, int y)`
- Purpose: Play sound with RTP (Real-Time Positioning?) using relative coordinates
- Inputs: Sound ID; x, y position
- Outputs/Return: Sound handle
- Side effects: Allocates voice; applies positioning
- Calls: Audio driver
- Notes: Simplified interface vs `SD_PlayPositionedSound`

### SD_Play3D
- Signature: `int SD_Play3D(int sndnum, int angle, int distance)`
- Purpose: Play sound with angle and distance (polar coordinate positioning)
- Inputs: Sound ID; angle (0–359?); distance (units)
- Outputs/Return: Sound handle
- Side effects: Allocates voice; applies pan/volume attenuation
- Calls: Audio driver
- Notes: Immersive spatial audio for monsters/explosions

### SD_PlayPitchedSound
- Signature: `int SD_PlayPitchedSound(int sndnum, int volume, int pitch)`
- Purpose: Play sound with custom pitch and volume
- Inputs: Sound ID; volume level; pitch value
- Outputs/Return: Sound handle
- Side effects: Allocates voice; applies pitch shift
- Calls: Audio driver
- Notes: Used for varied monster calls, weapon tones

### SD_PanPositionedSound
- Signature: `void SD_PanPositionedSound(int handle, int px, int py, int x, int y)`
- Purpose: Update panning of an active sound in real-time
- Inputs: Sound handle; panner point (px, py); listener position (x, y)
- Outputs/Return: None
- Side effects: Modifies pan/volume of running sound
- Calls: Audio driver
- Notes: Allows smooth positional audio updates as entities move

### SD_SetPan
- Signature: `void SD_SetPan(int handle, int vol, int left, int right)`
- Purpose: Directly set pan/volume for an active sound
- Inputs: Sound handle; volume; left/right pan levels
- Outputs/Return: None
- Side effects: Modifies audio stream
- Calls: Audio driver
- Notes: Raw stereo control

### SD_WaitSound
- Signature: `void SD_WaitSound(int handle)`
- Purpose: Block until a sound finishes playing
- Inputs: Sound handle
- Outputs/Return: None
- Side effects: Blocks caller
- Calls: Audio driver query loop
- Notes: Used for sync points (e.g., waiting for dialogue)

### SD_StopSound
- Signature: `void SD_StopSound(int handle)`
- Purpose: Stop a specific active sound
- Inputs: Sound handle
- Outputs/Return: None
- Side effects: Deallocates voice; stops audio output
- Calls: Audio driver
- Notes: Immediate stop

### SD_SoundActive
- Signature: `int SD_SoundActive(int handle)`
- Purpose: Query whether a sound is still playing
- Inputs: Sound handle
- Outputs/Return: Non-zero if playing, 0 if finished/stopped
- Side effects: None (query only)
- Calls: Audio driver query
- Notes: Used for UI feedback (e.g., waiting indicators)

### SD_StopAllSounds
- Signature: `void SD_StopAllSounds(void)`
- Purpose: Stop all active sounds immediately
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates all voices
- Calls: Audio driver
- Notes: Used for scene transitions, level changes

### SD_PreCacheSound
- Signature: `void SD_PreCacheSound(int num)`
- Purpose: Load sound into memory ahead of time
- Inputs: Sound ID
- Outputs/Return: None
- Side effects: Allocates memory; loads sound asset
- Calls: File/resource loader
- Notes: Prevents stutter during playback

### SD_PreCacheSoundGroup
- Signature: `void SD_PreCacheSoundGroup(int lo, int hi)`
- Purpose: Batch pre-cache multiple sounds
- Inputs: Low and high sound ID range (inclusive)
- Outputs/Return: None
- Side effects: Allocates memory; loads multiple assets
- Calls: File/resource loader
- Notes: Efficient precache for level/category (e.g., all enemy sounds)

### MU_Startup
- Signature: `int MU_Startup(boolean bombonerror)`
- Purpose: Initialize music subsystem
- Inputs: `bombonerror` – abort on error flag
- Outputs/Return: Status code
- Side effects: Sets up MIDI/music driver
- Calls: Music driver init
- Notes: Separate from SD_Startup; manages composition playback

### MU_Shutdown
- Signature: `void MU_Shutdown(void)`
- Purpose: Shut down music subsystem
- Inputs: None
- Outputs/Return: None
- Side effects: Stops music, deallocates resources
- Calls: Music driver shutdown
- Notes: Companion to MU_Startup

### MU_PlaySong
- Signature: `void MU_PlaySong(int num)`
- Purpose: Play a song by ID
- Inputs: Song type from `songtypes` enum
- Outputs/Return: None
- Side effects: Stops any current music; begins playback
- Calls: Music driver
- Notes: Direct play; respects loop flag set via `MUSIC_SetLoopFlag`

### MU_StartSong
- Signature: `void MU_StartSong(int songtype)`
- Purpose: Start a song by type (possibly with extra logic vs `MU_PlaySong`)
- Inputs: Song type
- Outputs/Return: None
- Side effects: Initiates music playback
- Calls: Music driver
- Notes: May differ from `MU_PlaySong` in context/initialization

### MU_StopSong
- Signature: `void MU_StopSong(void)`
- Purpose: Stop current music playback
- Inputs: None
- Outputs/Return: None
- Side effects: Halts music stream
- Calls: Music driver
- Notes: Immediate stop; does not fade

### MU_FadeIn
- Signature: `void MU_FadeIn(int num, int time)`
- Purpose: Fade music in over specified duration
- Inputs: Song type ID; fade time (milliseconds)
- Outputs/Return: None
- Side effects: Starts fade operation; begins playback
- Calls: Music driver fade
- Notes: Smooth volume ramp from 0

### MU_FadeOut
- Signature: `void MU_FadeOut(int time)`
- Purpose: Fade out current music over duration
- Inputs: Fade time (milliseconds)
- Outputs/Return: None
- Side effects: Starts fade; stops after fade completes
- Calls: Music driver fade
- Notes: Smooth volume ramp to 0; stops music at end

### MU_FadeToSong
- Signature: `void MU_FadeToSong(int num, int time)`
- Purpose: Fade to a new song (fade out old, fade in new)
- Inputs: New song type ID; fade time (milliseconds)
- Outputs/Return: None
- Side effects: Transitions music with crossfade effect
- Calls: Music driver
- Notes: Smooth transition between compositions

### MU_StoreSongPosition
- Signature: `void MU_StoreSongPosition(void)`
- Purpose: Save current playback position of music (for resume)
- Inputs: None
- Outputs/Return: None
- Side effects: Records position state
- Calls: Music driver query
- Notes: Used for pause/save-game restore

### MU_RestoreSongPosition
- Signature: `void MU_RestoreSongPosition(void)`
- Purpose: Resume from saved music position
- Inputs: None
- Outputs/Return: None
- Side effects: Seeks music to stored position
- Calls: Music driver seek
- Notes: Companion to `MU_StoreSongPosition`

### MU_GetSongPosition, MU_SetSongPosition
- Signature: `int MU_GetSongPosition(void)`, `void MU_SetSongPosition(int position)`
- Purpose: Query/set current music playback position
- Inputs: (setter) position offset
- Outputs/Return: (getter) position
- Side effects: (setter) Seeks music
- Calls: Music driver
- Notes: Real-time position control

### MU_GetSongNumber, MU_GetNumForType
- Signature: `int MU_GetSongNumber(void)`, `int MU_GetNumForType(int songtype)`
- Purpose: Query currently-playing song or map song type to ID
- Inputs: (latter) song type
- Outputs/Return: Song ID
- Side effects: None (query)
- Calls: Music driver query
- Notes: Used for UI, audio management

### MU_JukeBoxMenu
- Signature: `void MU_JukeBoxMenu(void)`
- Purpose: Display/handle jukebox menu for song selection
- Inputs: None
- Outputs/Return: None
- Side effects: Displays UI; may play selected song
- Calls: Music driver, UI system
- Notes: Dev/cheat feature for music selection

### MU_LoadMusic, MU_SaveMusic
- Signature: `void MU_LoadMusic(byte *buf, int size)`, `void MU_SaveMusic(byte **buf, int *size)`
- Purpose: Serialize/deserialize music state (save-game support)
- Inputs: (load) buffer + size; (save) buffer pointer, size pointer
- Outputs/Return: (save) populates buffer pointers
- Side effects: Allocates memory; loads state into music driver
- Calls: Memory allocator, music driver
- Notes: Used by save/load system

### MusicStarted
- Signature: `boolean MusicStarted(void)`
- Purpose: Query whether music subsystem has been initialized
- Inputs: None
- Outputs/Return: Boolean flag
- Side effects: None (query)
- Calls: None (direct state check)
- Notes: Safety check before playing music

## Control Flow Notes

**Initialization Phase:**
- `SD_SetupFXCard()` → `SD_Startup()` and `MU_Startup()` called at engine startup
- Sound and music systems operate independently after startup

**Gameplay Loop:**
- Game events (weapon fire, damage, item pickup, enemy action) trigger `SD_Play*()` calls
- Music transitions via `MU_PlaySong()`, `MU_FadeToSong()`, or via `song_*` enum triggers
- Positioned sounds (`SD_PlayPositionedSound`, `SD_Play3D`) update continuously as actors move
- `SD_PanPositionedSound()` keeps 3D audio in sync with world state

**Shutdown Phase:**
- `SD_StopAllSounds()` called before level transitions
- `SD_Shutdown()` and `MU_Shutdown()` called at game exit

**Save/Load:**
- `MU_StoreSongPosition()` / `MU_RestoreSongPosition()` preserve music state
- `MU_LoadMusic()` / `MU_SaveMusic()` serialize full music subsystem

## External Dependencies

- **music.h**: Declares `MUSIC_*` macro/function implementations; wrapped by `MU_*` functions
- **develop.h**: Debug/development feature flags (SOUNDTEST, PRECACHETEST, etc.)
- **sndcards.h**: Referenced by music.h; audio card abstraction layer (not included here)
- Audio device driver: Implied via `SD_SetupFXCard()`, hardware initialization
- Resource/file system: Implied by `SD_PreCacheSound()`, sound asset loading

**Notes:**
- 300+ `game_sounds` enum values indicate comprehensive audio asset library
- Macro wrappers (`MU_Continue()`, `MU_Pause()`, etc.) delegate to `MUSIC_*` functions for convenience
- Spatial audio (2D panning, 3D polar) suggests immersive sound design
- Support for multiple audio hardware (Adlib, MIDI, Sound Blaster) reflects 1990s hardware diversity
