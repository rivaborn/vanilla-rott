# audiolib/source/leetimb1.c — Enhanced Analysis

## Architectural Role

This file provides the **complete FM synthesis timbre database** for the AdLib audio engine, serving as the central lookup table for all melodic and percussion voices during MIDI playback. It bridges MIDI channel events (program changes, note-ons) to low-level AdLib hardware FM register configurations. The file is loaded once at engine initialization via `AL_RegisterTimbreBank()` and remains immutable during gameplay, making it a **read-heavy, initialization-time asset** rather than a runtime-modifiable instrument bank.

## Key Cross-References

### Incoming (who depends on this file)

- **`AL_RegisterTimbreBank()`** (al_midi.c) — Called during `AL_Init()` to register ADLIB_TimbreBank as the active timbre source
- **`AL_SetVoiceTimbre()`** (al_midi.c) — Looks up ADLIB_TimbreBank[index] to configure a voice's FM synthesis parameters
- **`AL_ProgramChange()`** (al_midi.c) — MIDI program-change events select timbre indices (0–255) for a channel
- **`AL_NoteOn()`** (al_midi.c) — Triggers voice allocation; voice setup uses ADLIB_TimbreBank to initialize FM operators
- **Percussion pathway**: MIDI drum notes (channels 120–127) → PercussionTable lookup → remapped timbre + key offset

### Outgoing (what this file depends on)

- None. This is a **pure-data resource** with no external dependencies or function calls.

## Design Patterns & Rationale

**1. Lookup Table Pattern**
- 256-entry ADLIB_TimbreBank allows direct timbre access via MIDI program number (0–127 GM, 128–255 extended)
- 128-entry PercussionTable maps MIDI drum notes to timbre + pitch offset pairs

**2. Sparse Percussion Mapping**
- Most PercussionTable entries are `{ 128, 0 }` (unmapped sentinel) — indices 0–35 and 88–127 are unimplemented
- Mapped drum sounds occupy indices ~36–87 (GM percussion range) with specific timbre + key values
- This suggests **selective drum support**: only commonly-used drum kits were FM-synthesized; others may fall back to sampled audio or silence

**3. Default Timbre Pattern**
- Repeated entries like `{ 33, 17 }, { 17, 0 }, { 163, 196 }, { 67, 34 }, { 2, 0 }, 13` appear dozens of times
- Indicates a **generic FM brass/string template** used as placeholder for unspecialized instruments
- Reduces file size while maintaining a valid timbre for every index

**4. Hardware Register Encoding**
- TIMBRE struct maps 1:1 to AdLib hardware FM register ranges (20h–35h for operator params, C0h–C8h for feedback)
- Each operator: `SAVEK` = attack-sustain-envelope key, `Level` = volume/attenuation, `Env1/Env2` = envelope decay/release rates, `Wave` = operator waveform
- Direct byte-pair layout suggests pre-computed hardware register values (no runtime conversion needed)

## Data Flow Through This File

```
MIDI Input (Program Change / Note On)
    ↓
AL_ProgramChange() / AL_NoteOn()
    ↓
[Look up timbre index in ADLIB_TimbreBank or PercussionTable]
    ↓
Extract TIMBRE struct (operator params + feedback)
    ↓
AL_SetVoiceTimbre() → write FM register values to AdLib hardware (20h–35h)
    ↓
Voice synthesis begins (modulation generates audio waveform)
```

**Percussion path:** Incoming MIDI note (36–87) → PercussionTable[note] yields `(timbre_idx, key_offset)` → lookup ADLIB_TimbreBank[timbre_idx] → apply key_offset to pitch calculation.

## Learning Notes

1. **FM Synthesis Era (1990s)**: This encoding reflects how composers and sound designers worked with real AdLib hardware—timbres were manually crafted as register byte sequences, not synthesized from high-level parameters.

2. **Instrumentation Philosophy**: The repeated default template and sparse drum mapping show **pragmatic resource allocation**—the engine shipped with a limited, hand-tuned instrument set rather than attempting comprehensive FM synthesis of all 128 GM instruments.

3. **Idiomatic vs. Modern**:
   - **Then (AdLib)**: Timbres hardcoded as raw hardware register values; loaded once, indexed on-the-fly
   - **Modern**: Wavetable/sample-based synthesis, dynamic timbre morphing, or procedurally generated FM parameters

4. **Hardware Constraints Visible in Design**: The 256-instrument limit reflects AdLib's 18-voice polyphony and the desire to fit a timbre bank in low conventional memory (a 256 × 6-byte table = ~1.5 KB).

## Potential Issues

- **Out-of-Bounds Sentinel**: PercussionTable entries with `Timbre == 128` are outside valid indices (0–255). Callers must special-case unmapped drums (likely silence or SFX fallback). No validation in this file.
- **Sparse Instrumentation**: Many entries in ADLIB_TimbreBank repeat the default template; game audio may sound generic if relying on unmapped instruments.
- **No Versioning/Metadata**: File is immutable constants; no flags to indicate which timbres are "official" vs. placeholder, or which reflect original game audio intent.
