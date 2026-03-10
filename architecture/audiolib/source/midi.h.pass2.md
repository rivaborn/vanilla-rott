# audiolib/source/midi.h — Enhanced Analysis

## Architectural Role

This header defines the MIDI playback subsystem's public interface, serving as the bridge between the game engine's music requests and the low-level synthesis backends (AL, AdLibFX, AWE32). The MIDI module acts as a **song controller and event router**: it parses MIDI data, maintains playback state (position, tempo, looping), and dispatches MIDI events to whichever synthesis backend is plugged in via the `midifuncs` callback structure. This design decouples the high-level game logic from hardware-specific synthesis.

## Key Cross-References

### Incoming (who depends on this file)
- **Game logic & menus** (via `rott/` modules): call `MIDI_PlaySong`, `MIDI_StopSong`, `MIDI_PauseSong` during menu transitions, level starts, and cinematics
- **Main game loop** (inferred): calls `MIDI_SongPlaying` to update HUD/state, queries position via `MIDI_GetSongPosition`
- **Initialization chain** (inferred): calls `MIDI_LockMemory`, `MIDI_LoadTimbres`, `MIDI_SetMidiFuncs` during startup

### Outgoing (what this file depends on)
- **Synthesis backends** (implemented in `audiolib/source/`):
  - `al_midi.c` (Audio Library): implements `AL_NoteOn`, `AL_NoteOff`, `AL_ProgramChange`, etc. — typically assigned to `midifuncs` callbacks
  - `adlibfx.c`: AdLib FM synthesis backend with `ADLIBFX_Play`, `ADLIBFX_SetCallBack`
  - `awe32.c`: EMU8000 wavetable backend with `AWE32_NoteOn`, `AWE32_PitchBend`, etc.
  - `blaster.c`: Sound Blaster hardware driver for DMA, sample playback (used indirectly by synthesis layers)
- **Resource managers** (inferred): 
  - DOS DPMI memory management (via `MIDI_LockMemory` / `MIDI_UnlockMemory`)
  - Timbre/patch banks (loaded via `MIDI_LoadTimbres`)
- **Global state**: `MIDI_PatchMap[128]` (extern, writable by calibration/config subsystems)

## Design Patterns & Rationale

**Callback/Function-Pointer Interface (`midifuncs`)**  
Rather than hard-coding calls to a specific synthesis backend, MIDI invokes function pointers. This is classic early-1990s hardware abstraction: the synthesis backend registers itself once (`MIDI_SetMidiFuncs`), and MIDI dispatches all note-on, control-change, pitch-bend events through the registered callbacks. **Why?** In 1994, synthesizer hardware varied widely (AdLib, Sound Canvas, Gravis UltraSound, EMU8000). This pattern allows shipping a single MIDI module that supports multiple cards without recompilation.

**Per-Channel Rerouting (`MIDI_RerouteMidiChannel`)**  
Allows overriding the default callback for a specific MIDI channel with a custom handler. **Why?** Enables per-channel effects (echo, reverb, compression) or routing (e.g., drums to a different voice pool).

**Global Patch Mapping (`MIDI_PatchMap`)**  
Maps MIDI program numbers (0–127) to hardware-specific timbre IDs. **Why?** MIDI uses standardized program numbers, but hardware timbres are numbered differently. The map allows remapping without modifying the MIDI file.

**Volume Layering**  
Separate global volume (`MIDI_SetVolume`), per-channel volume (`MIDI_SetUserChannelVolume`), and synthesis-backend volume (invoked via callback). **Why?** Game audio mixing: global for game pause/fade-out, per-channel for dynamic mixing of instruments.

**Tempo & Position Seeking**  
Supports three position units (ticks, milliseconds, measure-beat-tick). **Why?** Flexible integration: HUD can display position in musical units; seeking works in absolute time or relative ticks.

## Data Flow Through This File

```
Song File (binary MIDI data)
        ↓
  MIDI_PlaySong(unsigned char *song, loopflag)
        ↓
  [Parse MIDI events, setup playback state]
        ↓
  Main loop timer / MIDI engine service routine
        ↓
  [Advance to next MIDI event(s) at current tick]
        ↓
  MIDI_*funcs callbacks dispatched:
    - NoteOn(channel, key, velocity)
    - ProgramChange(channel, program)
    - ControlChange(channel, cc_num, value)
    - PitchBend(channel, lsb, msb)
    [etc.]
        ↓
  Synthesis Backend (AL_NoteOn → FM voice allocation → OPL3 register writes, etc.)
        ↓
  Audio Output (Sound Blaster DAC / hardware)
```

**Tempo & Position**: Maintained internally; game queries via `MIDI_GetSongPosition` to update HUD.  
**Volume**: Applied at MIDI layer (velocity scaling) and synthesis layer (voice amplitude), allowing independent control.

## Learning Notes

**1. DOS-era MIDI architecture:**
- DPMI memory locking (`MIDI_LockMemory/UnlockMemory`) reflects real-mode vs. protected-mode DOS constraints
- No native threading; MIDI engine called synchronously from timer interrupt or main loop
- Hard-coded patch map reflects limited synthesizer memory (pre-1990s wavetable cards had <1MB ROM)

**2. Callback-driven design over modern alternatives:**
- Modern game engines use event queues or message buses; this uses direct function pointers
- No async/await; synchronous callbacks invoked from the MIDI sequencer thread
- Reflects the era's preference for predictable, low-overhead dispatch

**3. Tempo represented internally** (inferred):
- `MIDI_SetTempo`, `MIDI_GetTempo` suggest internal tempo state (not encoded per-event in MIDI file)
- Game can speed up/slow down music dynamically (useful for action sequences)

**4. Multi-backend coexistence:**
- Codebase ships multiple synthesis backends (AL, AdLib, AWE32) but selects at **startup** via `MIDI_SetMidiFuncs`
- Not runtime switching; assumes hardware is fixed for a game session

**5. Idiomatic to 1990s game audio:**
- No sample-based music fallback (only MIDI synth)
- Volume control is MIDI-aware (velocity-based) and hardware-aware (callback-controlled)
- Seek-by-measure-beat-tick implies MIDIs were composed in standard DAWs with known structure

## Potential Issues

1. **No error recovery in playback**:
   - `MIDI_PlaySong` returns status, but ongoing playback errors (malformed MIDI event, synthesis backend crash) have no callback mechanism
   - If a synthesis backend fails silently, MIDI module has no way to detect/report

2. **Patch map mutation risk**:
   - `MIDI_PatchMap` is extern and writable; no synchronization between game logic and MIDI engine
   - Changing `MIDI_PatchMap` while a song is playing could cause audible glitches

3. **Channel rerouting isolation**:
   - `MIDI_RerouteMidiChannel` custom handlers receive only `(event, c1, c2)` — no context (e.g., song position)
   - Limits what custom handlers can do (e.g., cannot query current tempo)

4. **No resource cleanup on error**:
   - `MIDI_PlaySong` may partially initialize; if an error occurs mid-playback, `MIDI_StopSong` is required to clean up (implied contract, not enforced)
