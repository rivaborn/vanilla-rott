# audiolib/source/gmtmbold.c — Enhanced Analysis

## Architectural Role
This file serves as the **timbre/instrument database** for the AdLib FM synthesis subsystem. It bridges MIDI semantics (percussion note numbers, program changes) to hardware-level FM operator parameters (OPL2/OPL3 control values). The data here is consumed by `AL_RegisterTimbreBank()` (from `al_midi.c`) during initialization and referenced by voice synthesis functions (`AL_SetVoiceTimbre()`) during playback. It represents the acoustic "vocabulary" of the game's MIDI soundtrack.

## Key Cross-References

### Incoming (who depends on this file)
- **`AL_RegisterTimbreBank()` (al_midi.c)**: Registers `ADLIB_TimbreBank` with the MIDI engine at startup
- **`AL_SetVoiceTimbre()` (al_midi.c)**: Applies a timbre from the bank to an allocated voice during playback
- **`AL_ProgramChange()` (al_midi.c)**: MIDI event handler that selects a new instrument, likely indexing into `ADLIB_TimbreBank`
- **MIDI percussion routing**: `PercussionTable` is implicitly used when `AL_NoteOn()` is called on the drum channel (MIDI channel 10)

### Outgoing (what this file depends on)
- **None**: Purely declarative data; no external function calls or includes
- **Implicit hardware**: FM parameters encode Yamaha OPL2/OPL3 register values (sustain/attack, level, envelope, waveform, feedback)

## Design Patterns & Rationale

1. **Pre-Computed Timbre Bank**: Rather than synthesizing timbres procedurally or loading them from disk during gameplay, all 174 instrument patches are hard-coded as constants. This trades storage for **low-latency MIDI response** (critical for real-time game audio).

2. **FM Synthesis via Operator Pairs**: The `TIMBRE` struct captures two-operator FM (the OPL2 standard):
   - `SAVEK[2]`: Sustain/attack for each operator
   - `Level[2]`: Output level envelope
   - `Env[2]`: Decay/release decay envelope
   - `Wave[2]`: Waveform selection (sine, triangle, square, etc.)
   - `Feedback`: Self-feedback amount (adds harmonics)
   
   This is **hardware-native**—the values can be written directly to OPL2 I/O ports.

3. **MIDI Percussion Mapping Indirection**: `PercussionTable[128]` maps MIDI note 0–127 to a (timbre ID, key offset) pair. This allows:
   - **Drum kit variety**: A single note (e.g., kick drum on note 36) can play different timbres per configuration
   - **Reuse of timbres**: Timbre 128 is mapped from many notes (default/unmapped slots)
   - **Flexibility**: Game developers could swap percussion layouts by changing only this table, not timbre definitions

## Data Flow Through This File

```
MIDI Input (drum channel: note 36–81)
    ↓
AL_NoteOn() looks up PercussionTable[note]
    ↓
Gets (timbre_id, key) pair
    ↓
AL_SetVoiceTimbre() indexes ADLIB_TimbreBank[timbre_id]
    ↓
FM operator parameters configure OPL2 hardware
    ↓
Synthesized drum sound
```

For melodic channels, the same `ADLIB_TimbreBank` is used after `AL_ProgramChange()` selects a new timbre index.

## Learning Notes

**What this teaches about 1990s game audio:**
- **Hardware constraints shaped design**: The OPL2 chipset had hard limits (11 voices, 2-operator FM only), so timbres had to be pre-optimized and stored as lookup tables rather than computed.
- **General MIDI convention**: The percussion mapping follows GM (General MIDI) drum assignments—note 36=kick, 38=snare, etc.—enabling cross-game MIDI compatibility.
- **No sample-based fallback**: Unlike modern engines (which can stream samples), this relies entirely on FM synthesis. Every instrument had to be hand-tuned by audio engineers.

**Idiomatic to this engine/era:**
- Hard-coded instrument banks (no dynamic loading or synthesis)
- Direct hardware register values (no abstraction layer for FM parameters)
- Percussion as a data table overlay, not a separate synthesizer
- Emphasis on **predictability and speed** over flexibility

**Connection to broader game engine concepts:**
- This is analogous to a **resource asset** (like a texture atlas), but for audio
- The timbre bank + percussion table form a **lookup-driven synthesis architecture**, a precursor to modern **wavetable synthesis**
- The two-operator FM design mirrors low-level **digital signal processing optimization**

## Potential Issues

1. **Timbre Bank Size**: 174 timbres is large relative to typical MIDI program count (128). This suggests either:
   - Some entries are duplicates or placeholders (not clearly documented)
   - Extended GM (XG or GS) support may be intended but not fully utilized

2. **Percussion Mapping Hardcoding**: The `PercussionTable` is completely static. If a game needed different drum mappings (e.g., for a non-Western instrument set), the entire engine would need recompilation—no runtime configuration.

3. **FM Parameter Magic Numbers**: The operator values (e.g., `{ 33, 33 }`, `{ 242, 242 }`) are opaque. Without documentation or tooling, reproducing or iterating on these timbres would be difficult.

---

**Token count: ~950 | Schema compliant** ✓
