# audiolib/source/music.h — Enhanced Analysis

## Architectural Role
The MUSIC module is a **high-level MIDI music abstraction layer** that unifies playback across multiple 1990s-era sound card backends (Sound Blaster, AdLib, AWE32, etc.). It sits above device-specific MIDI drivers (like `al_midi.h`, `adlibfx.h`) and exposes a single, hardware-agnostic API to the game engine. The module handles song lifecycle (load, play, pause, seek, stop), volume management, and state preservation via context switching—enabling the game to seamlessly switch between background music tracks and game music during play.

## Key Cross-References

### Incoming (who depends on this file)
- **Game engine** (`rott/*` files): Likely calls `MUSIC_PlaySong`, `MUSIC_Pause`, `MUSIC_Continue`, and `MUSIC_StopSong` to manage background music and jingles during gameplay, menus, and cinematics
- **Game state managers** (`rt_game.c`, `cin_main.c`, menu system): Probably use `MUSIC_SetContext` to preserve/restore music state across game transitions
- **Main loop / engine core**: Indirectly consumes `MUSIC_FadeVolume` and `MUSIC_FadeActive` for cinematic or transition effects

### Outgoing (what this file depends on)
- **`sndcards.h`**: Provides the `SoundCard` enum (hardware type identifiers: Sound Blaster, AdLib, etc.)
- **Device drivers** (not explicitly shown in header, but inferred from function signatures):
  - `al_midi.c/h` (generic MIDI interface): Likely receives channel events via callbacks and handles actual MIDI note on/off, controller changes
  - `adlibfx.c/h` (AdLib synthesizer): Handles FM synthesis for sound cards without native MIDI
  - `blaster.c/h` (Sound Blaster PCM): Likely indirect dependency for sample playback
  - Hardware-specific backends (AWE32, GUS, etc.) via abstraction layer
- **Global state**: Reads/writes `MUSIC_ErrorCode` global for error reporting

## Design Patterns & Rationale

### **Hardware Abstraction / Facade**
The MUSIC module encapsulates complex device initialization (`MUSIC_Init` with `SoundCard` and `Address`) and exposes a unified interface. This isolates the game from 15+ incompatible sound card variants—a critical problem in the DOS/early Windows era.

### **Context Switching**
`MUSIC_SetContext` / `MUSIC_GetContext` allow multiple music states to be preserved and restored—likely supporting a layered approach where ambient music can pause while a boss theme plays, then resume. This was essential for early 1990s games with limited CPU budgets that couldn't mix multiple MIDI streams.

### **Callback-Based Channel Rerouting**
`MUSIC_RerouteMidiChannel` enables dynamic channel handling: the caller can intercept MIDI events and route them to custom synthesis, software effects, or specialized handling (e.g., percussion-only channels). This offers extensibility without modifying the core playback engine.

### **Structured Playback Position**
The `songposition` struct encodes position in three coordinate systems—ticks (raw MIDI clock), milliseconds (real-time), and measure/beat/tick (musical notation). This suggests the engine needed both precise frame-accurate seeking (for sync with cinematics) and human-readable notation (for level design tools).

### **Loop vs. Play-Once**
The macro pattern (`MUSIC_LoopSong`, `MUSIC_PlayOnce`) instead of boolean flags is idiomatic to Apogee's era—avoiding magic numbers in code and making intent explicit.

## Data Flow Through This File

```
Game Engine
    ↓
MUSIC_PlaySong(song_data, loopflag)
    ↓
[MUSIC module: parse MIDI, init playback state, set context]
    ↓
Device-specific backend (al_midi, adlibfx, etc.)
    ↓
Hardware (FM synthesis, MIDI port, wavetable synth)

Parallel flows:
- Volume commands (MUSIC_SetVolume) → all channels
- Per-channel volume (MUSIC_SetMidiChannelVolume) → specific channel
- Fade (MUSIC_FadeVolume) → gradual volume over time
- Position queries (MUSIC_GetSongPosition) ← device backend
- Context (MUSIC_SetContext) → pause/resume internal state
```

## Learning Notes

### **Multi-Device Era Design**
This header is a time capsule of 1990s PC audio chaos. A single game had to support:
- **AdLib** (FM synthesis, stereo, ~16 voices)
- **Sound Blaster** (PCM + OPL2/OPL3)
- **Gravis UltraSound** (wavetable sampler)
- **AWE32** (wavetable + effects)
- **Roland MT-32** (external MIDI device)

Each required different initialization sequences, MIDI channel assignments, and voice management. **MUSIC.H unifies this nightmare.**

### **No DSP-Level Control**
Unlike modern engines, there's no per-note effects (reverb, chorus, attack/release shaping) exposed here. All synthesis is delegated to hardware. The `MUSIC_RegisterTimbreBank` function suggests timbres are pre-created, pre-optimized for specific hardware—offline instrument design rather than runtime synthesis.

### **Synchronization with Gameplay**
The `milliseconds` and `measure/beat/tick` fields in `songposition` hint at **tight sync with cinematics and events**. Early 3D games needed MIDI to scrub to specific points for synchronized cutscenes—no streaming, no DSP, just raw MIDI playback with precise seeking.

### **No Streaming**
The `MUSIC_PlaySong( unsigned char *song, ... )` signature expects the entire MIDI file to be pre-loaded in memory. This contrasts with modern streaming approaches and reflects memory constraints of the era.

---

## Potential Issues

1. **No Resource Cleanup Visibility**: If `MUSIC_PlaySong` fails mid-init, unclear whether partially-initialized state is rolled back or left hanging. Error handling depends entirely on the implementation in MUSIC.C.

2. **Context Switching Semantics Unclear**: `MUSIC_SetContext` / `MUSIC_GetContext` lack documentation of what "context" means—is it just playback state, or does it include volume, position, etc.? This could lead to subtle bugs if callers misunderstand what's preserved.

3. **No Synchronization Primitives**: If the game runs asynchronous (interrupt-driven) MIDI playback, there's no explicit synchronization between position queries and playback updates. Race conditions possible if game reads `MUSIC_GetSongPosition` while hardware interrupt fires.

4. **Hardcoded Channel Rerouting**: `MUSIC_RerouteMidiChannel` accepts a raw function pointer. If the callback crashes or blocks, the entire music system freezes. No bounds checking or timeout.
