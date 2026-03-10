# rott/rt_sound.c

## File Purpose
Implements sound and music systems for the Rise of the Triad engine, including FX device initialization, 2D/3D sound playback with spatial audio, music song management with fading, and memory caching for audio resources.

## Core Responsibilities
- Initialize and manage sound card hardware (Sound Blaster, Adlib, UltraSound, etc.)
- Play sound effects with optional 3D positioning and pitch modulation
- Control sound panning and volume per-sound or globally
- Manage music playback with song selection, fading, and loop control
- Cache and pre-cache audio lumps for performance
- Store and restore music playback state for save/load game functionality
- Map sound and music to appropriate hardware device types based on configuration

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sound_t` | struct | Metadata and playback state for a sound effect (snds[], flags, priority, count, handle) |
| `song_t` | struct | Song metadata (loopflag, songtype, lumpname, songname) |
| `fx_device` | struct | FX card capabilities (MaxVoices, MaxSampleBits, MaxChannels) |
| `songposition` | struct | Current song position (tickposition, milliseconds, measure, beat, tick) |
| `game_sounds` | enum | All playable sound effect IDs (menu, player, weapons, actors, environment, remote) |
| `songtypes` | enum | Song context types (level, menu, boss, cinematic, title, etc.) |
| `fxtypes` | enum | Audio format types (fx_digital, fx_muse) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `soundstart` | int | static | Base lump offset for sound effects |
| `soundtype` | int | static | Current sound format (digital or MUSE-based) |
| `SD_Started` | int | global | Flag: FX system initialized |
| `PositionStored` | boolean | static | Flag: music position saved |
| `NumBadSounds` | int | static | Counter for failed sound playbacks (dev tracking) |
| `remotestart` | int | static | Base lump offset for remote command sounds |
| `SoundsRemapped` | boolean | static | Flag: digital sounds remapped to lump indices |
| `musicnums[11]` | int[] | static | Device ID array mapping music mode to hardware |
| `fxnums[11]` | int[] | static | Device ID array mapping FX mode to hardware |
| `currentsong` | byte* | static | Pointer to currently loaded song data |
| `MU_Started` | int | static | Flag: music system initialized |
| `lastsongnumber` | int | static | Index of currently playing song in rottsongs[] |
| `storedposition` | int | static | Saved music playback position (milliseconds) |
| `rottsongs[]` | song_t[] | static | Array of available songs (18 or 34 depending on SHAREWARE build) |

## Key Functions / Methods

### SD_Startup
- Signature: `int SD_Startup(boolean bombonerror)`
- Purpose: Initialize FX system, load sound lumps, set up audio hardware
- Inputs: `bombonerror` — if true, Error() on failure; if false, return status
- Outputs/Return: status code (0 on success, FX_* error on failure)
- Side effects: Sets `SD_Started`, `soundstart`, `soundtype`, `remotestart`; remaps digital sounds to WAD lump indices; calls FX_Init(), FX_SetCallBack()
- Calls: `W_GetNumForName()`, `FX_Init()`, `FX_SetCallBack()`, `FX_SetVolume()`, `SD_Shutdown()`, `Error()`
- Notes: Handles multiple sound card types with different sound table organization; applies IS8250 constraints (4 voices, 8-bit mono); remaps sounds only once per session

### SD_SetupFXCard
- Signature: `int SD_SetupFXCard(int *numvoices, int *numbits, int *numchannels)`
- Purpose: Query or initialize FX hardware without full startup
- Inputs: pointers to output variables for device capabilities
- Outputs/Return: status code; fills numvoices, numbits, numchannels on success
- Side effects: Calls FX_SetupCard() or FX_SetupSoundBlaster()
- Calls: `FX_SetupCard()`, `FX_SetupSoundBlaster()`, `SD_Shutdown()` if already running

### SD_PlayIt
- Signature: `int SD_PlayIt(int sndnum, int angle, int distance, int pitch)`
- Purpose: Internal dispatcher for all sound playback; loads sound and routes to VOC/WAV player
- Inputs: sound ID; 3D params (angle 0–2047, distance 0–255); pitch offset
- Outputs/Return: voice handle (FX_* error code if failure); 0 if no voice available
- Side effects: Increments sounds[sndnum].count; caches sound lump as PU_STATIC; may stop previous instance if not SD_WRITE flag; sets prevhandle and prevdistance
- Calls: `W_CacheLumpNum()`, `FX_VoiceAvailable()`, `FX_PlayVOC3D()`, `FX_PlayWAV3D()`, `SD_MakeCacheable()`, `FX_ErrorString()`
- Notes: Checks magic byte ('C' for VOC, else WAV); prioritizes closer new sounds over distant old ones

### SD_Play
- Signature: `int SD_Play(int sndnum)`
- Purpose: Play 2D sound at full volume with optional pitch variation
- Inputs: sound ID
- Outputs/Return: voice handle (0 if sound not okay or no voice)
- Side effects: Calls SD_PlayIt() with angle=0, distance=0
- Calls: `SD_SoundOkay()`, `PitchOffset()`, `SD_PlayIt()`

### SD_Play3D
- Signature: `int SD_Play3D(int sndnum, int angle, int distance)`
- Purpose: Play sound with 3D pan/volume based on angle and distance
- Inputs: sound ID; angle (0–2047, 0=front); distance (0–255, clamped)
- Outputs/Return: voice handle
- Side effects: Applies pitch shift unless SD_PITCHSHIFTOFF flag set
- Calls: `SD_SoundOkay()`, `PitchOffset()`, `SD_PlayIt()`

### SD_PlayPositionedSound
- Signature: `int SD_PlayPositionedSound(int sndnum, int px, int py, int x, int y)`
- Purpose: Play sound at absolute world coordinates relative to listener position
- Inputs: sound ID; listener (px, py); sound source (x, y) in game units
- Outputs/Return: voice handle
- Side effects: Converts to angle/distance and calls SD_PlayIt()
- Calls: `FindDistance()`, `atan2_appx()`, `SD_PlayIt()`

### SD_PlaySoundRTP
- Signature: `int SD_PlaySoundRTP(int sndnum, int x, int y)`
- Purpose: Play sound relative to player position (uses global `player` struct)
- Inputs: sound ID; sound source (x, y)
- Outputs/Return: voice handle
- Side effects: Subtracts player coords from sound coords; accounts for player facing angle
- Calls: Uses global `player->x`, `player->y`, `player->angle`; calls `FindDistance()`, `atan2_appx()`, `SD_PlayIt()`

### SD_PlayPitchedSound
- Signature: `int SD_PlayPitchedSound(int sndnum, int volume, int pitch)`
- Purpose: Play sound with explicit volume and pitch (distance is derived from volume)
- Inputs: sound ID; volume (0–255); pitch offset
- Outputs/Return: voice handle
- Side effects: Maps volume to distance: `distance = 255 - volume`
- Calls: `SD_SoundOkay()`, `SD_PlayIt()`

### SD_SetSoundPitch
- Signature: `void SD_SetSoundPitch(int sndnum, int pitch)`
- Purpose: Modify pitch of an active sound
- Inputs: sound/voice handle; pitch offset
- Side effects: Calls FX_SetPitch(); reports errors in dev builds
- Calls: `FX_SoundActive()`, `FX_SetPitch()`, `FX_ErrorString()`

### SD_PanRTP
- Signature: `void SD_PanRTP(int handle, int x, int y)`
- Purpose: Update 3D pan/volume of active sound relative to player
- Inputs: voice handle; sound position (x, y)
- Outputs/Return: void
- Side effects: Calls FX_Pan3D() with updated angle and distance
- Calls: `FX_SoundActive()`, `FindDistance()`, `atan2_appx()`, `FX_Pan3D()`

### SD_PanPositionedSound
- Signature: `void SD_PanPositionedSound(int handle, int px, int py, int x, int y)`
- Purpose: Update pan of sound given absolute listener and source positions
- Inputs: voice handle; listener (px, py); source (x, y)
- Side effects: Calls FX_Pan3D()
- Calls: `FX_Pan3D()`

### SD_SetPan
- Signature: `void SD_SetPan(int handle, int vol, int left, int right)`
- Purpose: Manually set stereo pan and volume
- Inputs: voice handle; overall volume; left and right channel volumes
- Side effects: Calls FX_SetPan()

### SD_StopSound / SD_StopAllSounds
- Signature: `void SD_StopSound(int handle)` / `void SD_StopAllSounds(void)`
- Purpose: Silence a single sound or all active sounds
- Side effects: Calls FX_StopSound() / FX_StopAllSounds()

### SD_SoundActive
- Signature: `int SD_SoundActive(int handle)`
- Purpose: Check if a voice is currently playing
- Outputs/Return: non-zero if active, 0 if not
- Calls: `FX_SoundActive()`

### SD_SoundOkay
- Signature: `boolean SD_SoundOkay(int sndnum)`
- Purpose: Validate sound before playback (checks init, bounds, SD_PLAYONCE flag)
- Outputs/Return: true if sound can play, false otherwise
- Side effects: Checks global SD_Started and sound flags
- Calls: `SoundOffset()`, `SD_SoundActive()`

### SD_MakeCacheable
- Signature: `void SD_MakeCacheable(unsigned long sndnum)`
- Purpose: Decrement sound usage count and re-cache if count reaches 0
- Inputs: sound ID (or -1 for no-op)
- Side effects: Called as FX_SetCallBack() when sound finishes; decrements sounds[sndnum].count
- Calls: `W_CacheLumpNum()`
- Notes: Used internally by FX system; implements reference counting for sound memory

### SD_PreCacheSound / SD_PreCacheSoundGroup
- Signature: `void SD_PreCacheSound(int num)` / `void SD_PreCacheSoundGroup(int lo, int hi)`
- Purpose: Force sound lump into cache at high priority for fast playback
- Side effects: Calls PreCacheLump() with priority derived from sound flags
- Calls: `SD_SoundOkay()`, `PreCacheLump()`

### MU_Startup
- Signature: `int MU_Startup(boolean bombonerror)`
- Purpose: Initialize music system and MIDI hardware
- Inputs: bombonerror — if true, Error() on failure
- Outputs/Return: status code (0 on success)
- Side effects: Sets MU_Started; calls MUSIC_Init(); initializes GUS if needed
- Calls: `MUSIC_Init()`, `MU_SetupGUSInitFile()`, `MU_SetVolume()`, `MU_StopSong()`, `MU_Shutdown()`
- Notes: May initialize FX system first if SoundBlaster/AWE32 card shared with FX

### MU_PlaySong
- Signature: `void MU_PlaySong(int num)`
- Purpose: Load and play a song by index in rottsongs[]
- Inputs: song index (0 to MAXSONGS-1)
- Side effects: Stops previous song; caches song lump; sets lastsongnumber; calls MUSIC_PlaySong() with loop flag
- Calls: `MU_StopSong()`, `W_CacheLumpName()`, `MUSIC_PlaySong()`, `MU_SetVolume()`, `Error()`

### MU_StartSong
- Signature: `void MU_StartSong(int songtype)`
- Purpose: Play context-aware song (e.g., level music varies by difficulty or Christmas)
- Inputs: song type enum (song_level, song_menu, etc.)
- Side effects: Looks up base song, adjusts index by context (Christmas, GetSongForLevel())
- Calls: `MU_GetNumForType()`, `IsChristmas()`, `GetSongForLevel()`, `MU_PlaySong()`

### MU_FadeToSong
- Signature: `void MU_FadeToSong(int num, int time)`
- Purpose: Fade out current music and fade in new song in specified time
- Inputs: song index; total transition time (milliseconds)
- Side effects: Fades out for time>>1, then fades in for time>>1
- Calls: `MU_FadeOut()`, `MU_FadeActive()`, `MU_FadeIn()`

### MU_FadeIn / MU_FadeOut
- Signature: `void MU_FadeIn(int num, int time)` / `void MU_FadeOut(int time)`
- Purpose: Fade in new song or fade out current music
- Side effects: Calls MUSIC_FadeVolume(); MU_FadeIn sets initial volume to 0 before playing
- Calls: `MU_PlaySong()`, `MUSIC_FadeVolume()`

### MU_StoreSongPosition / MU_RestoreSongPosition
- Signature: `void MU_StoreSongPosition(void)` / `void MU_RestoreSongPosition(void)`
- Purpose: Save/restore music playback position (for pausing in menus)
- Side effects: Sets PositionStored flag; copies storedposition from/to MUSIC_GetPosition()
- Calls: `MUSIC_GetPosition()`, `MUSIC_SetPosition()`

### MU_GetSongPosition / MU_SetSongPosition
- Signature: `int MU_GetSongPosition(void)` / `void MU_SetSongPosition(int position)`
- Purpose: Query or set playback position in milliseconds
- Calls: `MUSIC_GetPosition()`, `MUSIC_SetPosition()`

### MU_SaveMusic / MU_LoadMusic
- Signature: `void MU_SaveMusic(byte **buf, int *size)` / `void MU_LoadMusic(byte *buf, int size)`
- Purpose: Serialize/deserialize music state for save game (3 ints: song#, position, stored position)
- Side effects: If in menu, saves level song instead; restores song and position on load
- Calls: `SafeMalloc()`, `MU_PlaySong()`, `MU_SetSongPosition()`, `memcpy()`
- Notes: Handles special case where menu song is replaced with appropriate level song

### SoundNumber
- Signature: `int SoundNumber(int x)`
- Purpose: Internal helper to map sound ID to WAD lump number
- Inputs: sound index
- Outputs/Return: lump number = sounds[x].snds[soundtype] + soundstart (or remotestart for remote sounds)
- Notes: Routes remote sounds to separate base offset; called by most playback functions

## Control Flow Notes

**Initialization sequence:**
- `SD_Startup()` → `FX_Init()` + sound table remapping
- `MU_Startup()` → `MUSIC_Init()` + optional GUS setup
- Both can be called during game init or dynamically on config change

**Sound playback:**
- High-level API: `SD_Play()`, `SD_Play3D()`, `SD_PlaySoundRTP()` (user-facing)
- Mid-level: `SD_PlayPositionedSound()`, `SD_PlayPitchedSound()` (game-specific positioning)
- Low-level: `SD_PlayIt()` (dispatcher; loads lump, detects VOC/WAV, calls FX)
- Pan/pitch updates: `SD_PanRTP()`, `SD_SetSoundPitch()` (live control on active voices)

**Music playback:**
- Context-driven: `MU_StartSong(songtype)` selects song based on game state
- Direct: `MU_PlaySong(index)` for menu/jukebox
- Transitions: `MU_FadeToSong()` for seamless crossfades
- State persistence: `MU_SaveMusic()` / `MU_LoadMusic()` for save/load game

**Memory management:**
- Sounds cached as `PU_STATIC` during playback
- `SD_MakeCacheable()` called by FX system when sound finishes (reference counting)
- Songs cached as `PU_STATIC`; downgraded to `PU_CACHE` on stop
- Pre-caching via `SD_PreCacheSound()` for fast playback of critical sounds

## External Dependencies
- **Sound/FX system**: `fx_man.h` (FX_Init, FX_PlayVOC3D, FX_PlayWAV3D, FX_SetCallBack, FX_SetPitch, FX_Pan3D, FX_StopSound, FX_SoundActive)
- **Music system**: `music.h` (MUSIC_Init, MUSIC_PlaySong, MUSIC_FadeVolume, MUSIC_GetPosition, MUSIC_SetPosition, MUSIC_SongPlaying)
- **WAD/lump system**: `w_wad.h` (W_GetNumForName, W_GetNameForNum, W_CacheLumpNum, W_CacheLumpName, W_LumpLength)
- **Math**: `rt_util.h` (FindDistance, atan2_appx); inline `PitchOffset()` using RandomNumber()
- **Game state**: `rt_playr.h` (global `player` struct for relative sound positioning)
- **Utility**: `z_zone.h` (memory); `rt_rand.h` (RandomNumber); `rt_menu.h` (HandleMultiPageCustomMenu); `rt_main.h` (ticcount, Error, SoftError)
- **Configuration**: `rt_cfg.h` (FXMode, MusicMode, NumVoices, NumChannels, NumBits, FXvolume, MUvolume, stereoreversed, MidiAddress)
- **Development**: `develop.h` (DEVELOPMENT, SOUNDTEST flags for conditional logging)
