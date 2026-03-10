# rott/music.h — Enhanced Analysis

## Architectural Role
MUSIC.h is the **high-level song/music playback facade** in ROTT's audio subsystem. It abstracts hardware MIDI initialization and raw MIDI song playback behind a simple API, sitting between game code and lower-level MIDI drivers (AL_*, AWE32_*, BLASTER_* in audiolib). The file declares functions that initialize MIDI hardware, load/control song playback, manage channel volumes, and support real-time MIDI rerouting for custom event handling. This isolates game logic from hardware-specific MIDI details while supporting multiple sound cards (detected via `sndcards.h`).

## Key Cross-References

### Incoming (who depends on this file)
- **Game control logic** (rt_menu.h, rt_game.c): Calls MUSIC_Init/Shutdown during engine startup/teardown, MUSIC_PlaySong for level/menu music, MUSIC_SetVolume for options
- **Cinematic system** (cin_*.c): Likely uses MUSIC_PlaySong and MUSIC_GetSongPosition to sync cutscene timing with music
- **Likely callers from main.c**: Game initialization and cleanup

### Outgoing (what this file depends on)
- **sndcards.h**: Sound card enumeration (SoundCard parameter in MUSIC_Init)
- **audiolib low-level drivers**: The implementation (music.c) calls AL_Init, AL_ProgramChange, AL_NoteOn/Off, AL_RegisterTimbreBank for MIDI, and likely ADLIBFX_* for FM synth detection/initialization
- **BLASTER, AWE32, AL_MIDI hardware abstraction**: Initialized indirectly via MUSIC_Init's sound card routing

## Design Patterns & Rationale

**Subsystem Facade Pattern**: MUSIC.h wraps complex MIDI/hardware initialization (SoundCard + Address parameters) into a single Init call. Game code never touches AL_* or BLASTER_* directly.

**Error Code Convention**: Global `MUSIC_ErrorCode` + function return values following MUSIC_ERRORS enum (negative=error, 0=ok, positive=warnings). This is typical DOS-era error handling—caller must check both return value and global state.

**Callback-driven MIDI rerouting** (`MUSIC_RerouteMidiChannel`): Suggests interrupt or polled-interrupt architecture where MIDI events flow through custom callbacks at runtime. This enables real-time MIDI effects (e.g., monster sounds via MIDI remapping).

**Context switching** (`MUSIC_SetContext` / `MUSIC_GetContext`): Likely manages multiple MIDI channels or groups (e.g., background music vs. MIDI-routed SFX). Context may affect which channels respond to volume commands or routing changes.

**Why this structure**: Late 1990s DOS/protected-mode limitation: Hardware MIDI is interrupt-driven, so the subsystem must track state globally and allow runtime rerouting. Fade effects and tick-based positioning require either interrupt handlers or a polled update function (not visible in header—likely called from main game loop).

## Data Flow Through This File

```
Game Code
  ↓
MUSIC_Init(SoundCard, Addr) → [initialize AL_Init, detect BLASTER/ADLIB/AWE32]
  ↓
MUSIC_PlaySong(raw_midi_data, loopflag) → [load MIDI into AL_ subsystem, start playback]
  ↓
Per-frame:
  ├─ MUSIC_SetVolume() / MUSIC_FadeVolume() → volume state queued
  ├─ MUSIC_SetSongPosition() / seek → AL_* position updates
  └─ [Hardware interrupt or polled update processes MIDI events + fade interpolation]
  ↓
MUSIC_GetSongPosition() ← [query current tick/measure/beat]
  ↓
MUSIC_Pause / MUSIC_Continue → [pause MIDI stream without stopping]
  ↓
MUSIC_StopSong() / MUSIC_Shutdown() → [cleanup]
```

Raw MIDI song data → AL_* driver → hardware (BLASTER DSP, ADLIB OPL3, or MPU-401 UART) → audio out.

## Learning Notes

**Idiomatic to 1990s DOS game engines**:
- Explicit hardware addressing (I/O port address in MUSIC_Init) — modern engines assume OS abstracts this
- Global error code + per-function return (redundant but defensive)
- Tick-based MIDI sequencing with manual seek support (no built-in looping in MIDI protocol)
- `cdecl` calling convention explicit in callback signature — C++ or stdcall would hide this

**MIDI design choice**: MIDI is hardware-agnostic (notes, controllers, program changes) but requires a **driver per sound card type** (BLASTER DSP, ADLIB, FM, MPU-401 UART, AWE32). The subsystem abstracts this via `sndcards.h` + low-level AL_* drivers.

**Timbre bank registration**: `MUSIC_RegisterTimbreBank()` suggests custom instrument patches for FM synthesis or soundfont-like bank swapping. This is FM-synth specific (ADLIB/OPL3).

**Position/timing model**: Separation of tick, millisecond, and measure/beat/tick suggests:
- Internal tick clock (MIDI standard: 24 ticks per quarter note, but customizable)
- Tempo-dependent conversion to real time
- UI-friendly measure/beat/tick display

## Potential Issues

1. **No visible tick/update function**: If fade effects and MIDI playback are interrupt-driven, there's implicit timing dependencies. If polled, the game loop must call an unlisted MUSIC_Update() or similar. The header doesn't expose this—risk of desync if callers forget to poll.

2. **Context semantics unclear**: MUSIC_SetContext with no visible documentation on what "context" means. Could collide with other subsystem contexts if not coordinated.

3. **Reroute callback design**: `MUSIC_RerouteMidiChannel` callback signature `(int event, int c1, int c2)` is opaque. No clear contract on event values (MIDI status byte?) or parameter meanings. Callers must know MIDI protocol internals.

4. **Global MUSIC_ErrorCode**: Not thread-safe (relevant if network code or async loading reads it concurrently). Likely not an issue in single-threaded DOS, but fragile if ported.
