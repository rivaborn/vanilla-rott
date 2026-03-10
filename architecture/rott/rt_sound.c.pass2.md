# rott/rt_sound.c — Enhanced Analysis

## Architectural Role

This file implements ROTT's unified audio abstraction layer, mediating between game logic and a fragmented mid-90s hardware ecosystem. It provides high-level APIs (`SD_Play*`, `MU_*`) that hide multiple sound card backends (Sound Blaster, Adlib, UltraSound, etc.) and dual audio modes (digital PCM vs FM synthesis). The file also manages sound resource lifecycle through the WAD caching system and orchestrates 3D spatial audio by converting world coordinates to angle/distance pairs that the FX driver consumes.

## Key Cross-References

### Incoming (who depends on this file)
- **Game logic** (`rt_actor.c`, `rt_playr.c`, `rt_game.c`, cin_*.c): Calls `SD_Play*()` for enemy vocalisations, weapon fire, footsteps; calls `MU_*()` for context-aware music (menu, level, boss, cinematics)
- **Menu/UI** (`rt_menu.h`): Calls `MU_StorePosition()` / `MU_RestorePosition()` to pause music during menu overlays; uses `HandleMultiPageCustomMenu()` for sound settings
- **Game state** (`rt_main.c`): Initialization calls `SD_Startup()` / `MU_Startup()` during boot; shutdown calls `SD_Shutdown()` / `MU_Shutdown()`
- **Serialization** (save/load): Calls `MU_SaveMusic()` / `MU_LoadMusic()` to preserve song selection and playback position across save slots

### Outgoing (what this file depends on)
- **Audio drivers** (`fx_man.h`, `music.h`): Dispatches to `FX_Init()`, `FX_PlayVOC3D()`, `FX_PlayWAV3D()`, `FX_Pan3D()`, `FX_SetPitch()`, `MUSIC_PlaySong()`, `MUSIC_FadeVolume()`
- **Resource system** (`w_wad.h`): Looks up lump numbers via `W_GetNumForName()`, loads sound/song buffers via `W_CacheLumpNum()` / `W_CacheLumpName()`
- **Game state** (`rt_playr.h`): Reads global `player` struct (position, angle) for relative 3D sound positioning
- **Math utilities** (`rt_util.h`): Uses `FindDistance()` and `atan2_appx()` for world-to-polar conversion
- **Configuration** (`rt_cfg.h`): Reads globals `FXMode`, `MusicMode`, `NumVoices`, `NumBits`, `NumChannels`, `FXvolume`, `MUvolume`, `stereoreversed`, `MidiAddress`
- **Development** (`develop.h`): Conditional error logging via `DEVELOPMENT` and `SOUNDTEST` flags

## Design Patterns & Rationale

**Hardware Abstraction via Lookup Tables**  
`musicnums[11]` and `fxnums[11]` map configuration mode indices to device IDs. This enables compile-time or runtime device selection without if-chains. Notably, `fxnums` includes PC speaker (fallback) while `musicnums` includes AdLib/GM options.

**Callback-Driven Resource Management**  
`FX_SetCallBack(SD_MakeCacheable)` invokes `SD_MakeCacheable()` when hardware finishes playing a sound. This implements reference-counted cleanup without polling—idiomatic for ISR-era code. The `sounds[].count` field tracks overlapping playback instances (SD_PLAYONCE flag prevents multiple simultaneous plays).

**Lazy Remapping for Format Abstraction**  
Digital sounds are loaded from the sound table with indices like `sounds[x].snds[fx_digital]`. On first `SD_Startup()` with digital hardware, these indices are remapped in-place to WAD lump numbers. Adlib/PC sounds skip remapping (static offset). This amortizes the conversion cost.

**Dual-Mode Audio**  
`soundtype` selects between `fx_digital` (PCM, 11 kHz) and `fx_muse` (FM/MIDI synthesis). Different sound tables are indexed accordingly. Remote multiplayer commands use a separate lump base (`remotestart`), isolated from local sounds.

**Software 3D Panning**  
No hardware 3D support; instead, world coordinates are converted to angle (0–2047, 11-bit fine angle) and distance (0–255, clamped). These are passed to `FX_PlayVOC3D()` / `FX_Pan3D()`, which the driver translates to stereo panning and volume attenuation.

## Data Flow Through This File

```
[Game Code]
    ↓
SD_Play() / SD_Play3D() / SD_PlaySoundRTP() [validation + pitch]
    ↓
SD_PlayIt() [dispatcher]
    ├→ SoundNumber() [maps sound ID to lump number]
    ├→ W_CacheLumpNum() [load lump as PU_STATIC]
    ├→ Detect format (magic byte: 'C'=VOC, else WAV)
    └→ FX_PlayVOC3D() or FX_PlayWAV3D() [hardware dispatch]
        ↓
[Hardware ISR]
    ↓
FX callback → SD_MakeCacheable() [decrement count, re-cache if 0]
    ↓
W_CacheLumpNum() [mark PU_CACHE for purging]
```

**Music Flow:**  
```
[Game Context Change]
    ↓
MU_StartSong(song_level) → look up song via context (Christmas? difficulty?) → MU_PlaySong()
    ↓
W_CacheLumpName() [cache song as PU_STATIC]
    ↓
MUSIC_PlaySong() [driver queues song, sets loopflag]
    ↓
MU_FadeToSong() [optional fade: FX volume ramp via MUSIC_FadeVolume()]
```

## Learning Notes

**For students of 90s game audio:**
- **No dynamic allocation:** Sound metadata lives in a pre-allocated `sound_t` array. Scaling to more sounds requires recompiling.
- **Hardware-centric design:** Code paths diverge by device (Sound Blaster stereo setup, Adlib MIDI routing, PC speaker fallback).
- **Software 3D:** Engines of this era lacked hardware 3D audio; angle/distance are computed and re-computed per frame for moving sources.
- **Callback discipline:** ISR safety enforced by limiting callback work to reference-count decrement and cache marking.

**Contrast with modern engines (Unity, Unreal, custom modern SDKs):**
- Modern: Spatial audio via `AudioListener` + 3D transform; hardware/spatialization handled transparently.
- ROTT: Manual listener/source coordinate conversion; format detection at runtime; device-specific quirks (IS8250 constraints).

**Idiomatic patterns:**
- **Lump offset remapping:** Unique to WAD-based engines; maps abstract resource IDs to physical lump indices.
- **Fine-angle representation (FINEANGLES bits):** Fixed-point angle math common in raycasters; avoids floating-point per sound.
- **Shareware/registered build split:** `#if SHAREWARE` changes song tables and feature set; compile-time configuration.

## Potential Issues

1. **Hardcoded array sizes**: `sounds[]` is fixed-size (`MAXSOUNDS`), determined at compile time. No runtime growth; music limited to ~18–34 songs depending on build. Extending requires recompilation and changes to `snd_reg.h` / `snd_shar.h`.

2. **Music position loss on failure**: `MU_SaveMusic()` saves position only if `MU_Started`. If music fails to initialize, saved games will lose music context on reload.

3. **Concurrent sound overflow silent failure**: If all voices are busy, `SD_PlayIt()` returns 0 (no error code distinction between "out of voices" and "sound invalid"). Calling code cannot distinguish or retry.

4. **No pan boundary checks**: `SD_Pan3D()` does not clamp angle or distance; caller responsibility to validate. Out-of-range values may cause undefined FX driver behavior.

5. **Global config read race**: `SD_Startup()` reads globals like `FXMode`, `NumVoices` without locking. In a multi-threaded context or if config changes mid-game, behavior is undefined.

---

*This analysis reflects design decisions typical of mid-1990s PC game engines: hardware abstraction without hardware abstraction layers, callback-driven resource management, and software-based spatial audio.*
