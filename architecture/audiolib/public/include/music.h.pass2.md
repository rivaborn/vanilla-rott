# audiolib/public/include/music.h — Enhanced Analysis

## Architectural Role
The MUSIC module serves as the **high-level music playback abstraction** for the game engine, sitting above driver-specific MIDI implementations (AL_MIDI, BLASTER FM, AWE32, GUS). It provides a unified interface for MIDI song control regardless of hardware, abstracting away driver complexity while exposing game-relevant capabilities like looping, volume fading, and context-based multiplexing. This is a classic **facade pattern** that shields the game layer from audiolib's internal driver chaos.

## Key Cross-References

### Incoming (who depends on this file)
- **Game engine** (rott/*.c): Main game loop calls MUSIC_PlaySong, MUSIC_SetVolume, MUSIC_Pause/Continue for background music management
- **Menu system** (CP_SoundSetup, etc.): Configuration screens call MUSIC_Init/Shutdown and channel volume controls
- **Cinematic system** (cin_*.c): Likely uses MUSIC_SetSongPosition and MUSIC_FadeVolume for score timing in cutscenes
- **Net code** (rt_net.c): May synchronize music context across multiplayer sessions

### Outgoing (what this file depends on)
- **AL_MIDI driver** (al_midi.c): Core MIDI playback; MUSIC_PlaySong, channel routing, timbre loading all delegate here
- **BLASTER DSP** (blaster.c): FM synthesis and sample playback for SoundBlaster cards
- **AWE32 driver** (awe32.c): Alternative wavetable synth support
- **sndcards.h**: Sound card type enum (passed to MUSIC_Init to select appropriate driver)
- **Global MUSIC_ErrorCode**: Modified by all functions; read by caller to detect errors

## Design Patterns & Rationale

| Pattern | Evidence | Why? |
|---------|----------|------|
| **Facade/Adapter** | MUSIC wraps AL_MIDI, BLASTER, AWE32 under unified interface | Abstract hardware diversity in 1994 (ISA MIDI cards had wildly different APIs) |
| **Error Code Return** | MUSIC_ErrorCode global + return codes | DOS-era convention; no exceptions; allows C89 compatibility |
| **Callback Injection** | MUSIC_RerouteMidiChannel(channel, function_ptr) | Enable runtime MIDI processing (e.g., percussion layering, pitch shifting) without modifying playback loop |
| **Temporal Abstractions** | Three seek methods (tick, millisecond, measure/beat/tick) | Game scenarios vary: script timing needs ticks, UI seeks by time, musicians think in measures |
| **State-Machine Context** | MUSIC_SetContext/GetContext | Separate music tracks per game state (menu ≠ combat ≠ boss); multiplexing without full pause |
| **Deferred Fade** | MUSIC_FadeVolume → poll MUSIC_FadeActive() | Non-blocking; game loop handles fade updates; no blocking I/O |

## Data Flow Through This File

```
Input: MIDI song data (unsigned char *song, raw format)
    ↓
MUSIC_Init(soundcard, address) → selects driver, initializes AL_MIDI/BLASTER
    ↓
MUSIC_PlaySong(song, loopflag) → parses MIDI, schedules notes via AL_MIDI
    ↓
Game Loop (each frame):
  - MUSIC_SetVolume / MUSIC_FadeVolume → channel volume → AL_MIDI → hardware
  - MUSIC_SetSongTick/Time/Position → seek in MIDI stream → AL_MIDI
  - MUSIC_SongPlaying / MUSIC_GetSongPosition → query playback state
  - MUSIC_RerouteMidiChannel callback → intercept notes → custom handler
    ↓
Output: MIDI note-on/off events → AL_MIDI → BLASTER/AWE32 → DAC/synth
```

Key state machine:
- **Uninitialized** → MUSIC_Init → **Ready**
- **Ready** → MUSIC_PlaySong → **Playing**
- **Playing** ← MUSIC_Pause → **Paused** ← MUSIC_Continue → **Playing**
- **Playing** → MUSIC_StopSong → **Ready**
- **Playing** → MUSIC_Shutdown → **Uninitialized**

## Learning Notes

**Idiomatic Patterns from This Era:**
1. **Global error code**: Pre-C++ exception handling; standard in DOS/early Win3.1 games
2. **Unsigned char pointers for data**: No strong typing; caller responsible for format knowledge (here: MIDI)
3. **Cdecl calling convention**: Named explicitly in callback signature (`int cdecl (*function)`) because stdcall was default on 16-bit Windows
4. **Measure/beat/tick notation**: Shows musical domain knowledge; borrowed from sequencer/DAW terminology
5. **No resource cleanup via GC**: Manual MUSIC_Init/Shutdown pairs; common in C of this era

**Modern Comparisons:**
- Similar to web audio APIs (Web Audio Context) but with explicit state transitions instead of async
- Timbre bank registration resembles VST plugin bank loading
- Context switching resembles bus-based mixing in DAWs (separate bus per scene)
- Callback interception is an early form of middleware/plugin pattern

**Key Insight for Game Developers:**
This header reveals that **music in 1994 games was deeply integrated with driver management**. Unlike modern engines (Unity, Unreal) where you load a .ogg and play it, here the game must know whether the sound card is FM, wavetable, or sampled, and different timbres might sound wildly different. MUSIC abstracts this but doesn't eliminate it—the game still selects a sound card upfront.

## Potential Issues

1. **No concurrent song support**: Only one song plays at a time (MUSIC_PlaySong stops current). Multiplayer or multi-context scenarios must alternate songs via MUSIC_SetContext.
2. **Callback rerouting assumes cooperative scheduling**: MUSIC_RerouteMidiChannel callback runs in MIDI ISR context (likely); custom handlers must be very fast. No documentation visible for reentrance safety.
3. **Seek precision loss**: Converting between ticks, milliseconds, and measure/beat/tick invites rounding errors; no visible rounding specification in header.
4. **Global error state**: MUSIC_ErrorCode is not thread-safe; corrupts if multiple callers check simultaneously (unlikely in single-threaded DOS but poor practice).
5. **Fade granularity unclear**: MUSIC_FadeVolume(tovolume, milliseconds) doesn't specify interpolation curve (linear? exponential?) or frame-rate dependency.
