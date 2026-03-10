# rott/rt_sound.h — Enhanced Analysis

## Architectural Role

`rt_sound.h` is the **public audio facade** for Rise of the Triad's engine, bridging all game systems (actors, weapons, UI, cinematics, environment) to a hardware-abstracted audio stack. It decouples game code from device specifics (Sound Blaster, Adlib, MIDI, PC Speaker) via a single high-level API, enabling the same binary to work across 1990s consumer audio hardware without recompilation. The dual API (`SD_*` for effects, `MU_*` for music) reflects architectural separation: effects use pooled voices with real-time control; music is composition-driven via a synth backend (MUSE or MIDI).

## Key Cross-References

### Incoming (who depends on this)
- **Game systems**: `rt_actor.c` (enemy sounds), `rt_playr.c` (player damage/weapon fire), `rt_battl.c` (battle events)
- **Environment**: `rt_stat.c` (animated objects), `rt_door.c` (doors/triggers), `rt_map.c` (level events)
- **UI/menus**: `rt_menu.h` (UI feedback sounds)
- **Cinematics**: `cin_actr.c`, `cin_glob.c` (scripted audio)
- **Core engine**: `rt_main.c`, `engine.c` (init/shutdown)
- **Networking**: `rt_net.c` (game events across network)

### Outgoing (what this file calls)
- **Low-level drivers** (in `audiolib/source/`):
  - `blaster.c` (Sound Blaster DMA, DSP)
  - `adlibfx.c` (FM synth, 9 melodic + 5 drum channels)
  - `al_midi.c` (MIDI voice allocation, note on/off)
  - `awe32.c` (wavetable synth, pitch bend, aftertouch)
  - `gus.c` (Gravis UltraSound RAM/voice management)
- **Wrapper**: `music.h` (MUSE synth or MIDI routing)

## Design Patterns & Rationale

**Hardware Abstraction Layer (HAL)**: The `ASSTypes` enum + `SD_SetupFXCard()` + `SD_Startup()` trio implement a detector pattern—game queries available hardware, then all subsequent calls route to the selected driver (Blaster vs. Adlib vs. MIDI) via function pointers in the implementation (not visible here). This avoids branching in gameplay code.

**Voice Pooling**: `SD_PreCacheSoundGroup()` pre-loads sounds to avoid allocation stalls. Voice count negotiated in `SD_SetupFXCard()` reflects real hardware constraints (e.g., Sound Blaster might offer 16 voices, Adlib only 9). Game must respect this limit or sounds will drop.

**Positioned Audio Duality**: 
- `SD_PlayPositionedSound(px, py, x, y)` — Cartesian listener + source (2D world)
- `SD_Play3D(angle, distance)` — Relative polar (source relative to listener)
Both feed into pan/volume calculations in the driver layer. This suggests the game has 2D sector/door-based world, not true 3D.

**Music/SFX Separation**: Music (MUSE synth or MIDI) occupies dedicated channels; effects use independent voice pool. This avoids SFX monopolizing MIDI channels needed for composition.

**Real-Time Control**: Returned sound handles + `SD_PanPositionedSound()` + `SD_SetPan()` allow dynamic re-panning—critical for 3D immersion as enemies/explosions move. This presupposes a game loop that updates positioned sounds per frame.

## Data Flow Through This File

**Level Load Path**:  
Game loads level → `SD_PreCacheSoundGroup(enemy_sounds)` → sounds buffered to RAM (Blaster) or preset in MIDI (Adlib) → reduced latency on playback

**Positioned SFX Path**:  
Enemy fires weapon → `SD_Play3D(ENEMYFIRE_SND, angle_to_player, distance)` → driver converts polar to pan/volume → allocates voice → outputs to speakers → next frame `SD_PanPositionedSound()` updates pan as listener/source move

**Music Transition Path**:  
Level start → `MU_FadeOut(500ms)` → `MU_FadeToSong(level_music, 500ms)` → MUSE synth or MIDI driver fades out old, fades in new (smooth UX)

**Shutdown Path**:  
Game exit → `SD_StopAllSounds()` → `SD_Shutdown()` → releases DMA/memory/IRQ handlers; `MU_Shutdown()` → stops synth/MIDI

## Learning Notes

**What studying this teaches:**
- **1990s audio paradox**: Hardware was chaotic (PC speaker to AWE32 synth to MIDI), so abstraction was essential. Modern engines face similar multiplicity (WebAudio, Native Audio, platform SDKs).
- **Voice budgets**: Developers must think in terms of "how many simultaneous sounds?" not "play all sounds." RTS/Starcraft et al. had explicit voice limits (8–16 total).
- **Precaching disciplines**: Frame hitches from disk I/O are unacceptable; sounds must preload.
- **Spatial audio without math**: The engine hides trig (angle/distance → pan/volume) behind simple APIs.
- **Music as composition, not stream**: MUSE is synthesized real-time (low bandwidth), not streamed samples (needed for 1990s modems).

**Era-specific idioms**:
- Enum-based sound IDs (no dynamic asset loading; ~300 enum values imply massive asset database)
- Hardware explicit in public API (user must select Adlib/MIDI/Blaster); no auto-detection
- Macro wrappers (`MU_Continue()`, `MU_Pause()`) wrapping lower-level MUSIC_* functions (separation of music driver from game layer)

**Modern divergence**:
- Today: FMOD/Wwise, Dolby Atmos, automatic spatial (HRTF)
- Then: Manual pan/volume per sound; hardware must be selected at install time

## Potential Issues

1. **Voice starvation**: `SD_SetupFXCard()` negotiates max voices, but no fallback if game requests more than available. Risk: sounds are silently dropped.
2. **Hardware selection UI**: `ASSTypes` enum assumes user/installer picks card; no API shown for graceful fallback (e.g., if AWE32 unavailable, use Blaster?).
3. **Spatial audio rounding**: Integer coordinates in `SD_PlayPositionedSound()` and polar in `SD_Play3D()` may cause audible stepping/pops if listener moves smoothly.
4. **Music/SFX priority**: MUSE synth with 9 melodic channels + music can be starved by SFX. No documented priority system.
5. **Save state fragility**: `MU_StoreSongPosition()` saves MIDI position, but if hardware was switched or tempo changed, restore may fail or sound wrong.
