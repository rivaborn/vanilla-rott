# audiolib/source/gmtemp.c — Enhanced Analysis

## Architectural Role

This file serves as the **global timbre/instrument bank** for the AdLib MIDI subsystem. It bridges game sound requests (via MIDI program changes) to OPL2 FM synthesis hardware register values. The 256 pre-computed patches enable the music/SFX system to instantly configure FM operators without runtime synthesis parameter calculation—critical for 1990s DOS performance constraints.

## Key Cross-References

### Incoming (who depends on this file)
- **`al_midi.c` / `AL_RegisterTimbreBank`**: Registers `ADLIB_TimbreBank` at initialization (likely via a header export or direct extern reference)
- **`al_midi.c` / `AL_SetVoiceTimbre`**: Selects a timbre index during note-on playback (indexes into this bank)
- **`al_midi.c` / `AL_ProgramChange`**: Maps incoming MIDI program-change messages to timbre indices in this bank
- **`adlibfx.c`**: Sound effects system may also reference timbres for one-shot FX playback

### Outgoing (what this file depends on)
- **None**: This is a pure data file with no function calls or external symbol dependencies. No outgoing edges.

## Design Patterns & Rationale

**Pattern: Pre-baked Lookup Table**
- Timbres are **static at compile time**, not computed at runtime
- Each entry is a hand-tuned FM synthesis patch (likely created with 3rd-party AdLib editor software)
- Eliminates expensive parameter calculations during MIDI playback or voice allocation
- Typical for DOS-era engines: trade memory for CPU speed

**TIMBRE Structure Layout → OPL2 Hardware Mapping**
| Field | OPL Purpose | Notes |
|-------|-------------|-------|
| `SAVEK[2]` | Key Scale / Sustain Level (one per operator) | Controls how pitch affects decay |
| `Level[2]` | Output level (one per operator) | Volume/amplitude per operator |
| `Env1[2]` | Attack/Decay/Sustain/Release envelope (modulator side) | Envelope shape |
| `Env2[2]` | Attack/Decay/Sustain/Release envelope (carrier side) | Envelope shape |
| `Wave[2]` | Waveform select per operator (sine, half-sine, etc.) | Timbre character |
| `Feedback` | Feedback amount + algorithm (11 bits total) | Controls modulation topology |
| `Transpose` | Pitch offset (semitones) | Global transposition |

The structure directly encodes two-operator FM synthesis in OPL2's register semantics—no intermediate conversion needed.

## Data Flow Through This File

```
MIDI Input (note-on or program-change)
    ↓
al_midi.c: AL_ProgramChange() selects index into ADLIB_TimbreBank
    ↓
al_midi.c: AL_SetVoiceTimbre(voiceID, timbres[index]) 
    ↓
OPL2 hardware registers written (SAVEK, Level, Env1, Env2, Wave, Feedback, Transpose)
    ↓
Sound output (FM synthesis)
```

**State**: Immutable at runtime. Timbres never change once loaded.

## Learning Notes

**What this teaches about the engine:**
1. **OPL2 FM synthesis era** (early-to-mid 1990s): Two-operator FM, no wavetable, no sampling
2. **Offline patch design**: Timbres were created externally, not synthesized on-the-fly
3. **Hardware-mapped structures**: The C struct directly mirrors chip register layout—no abstraction layer
4. **256-instrument MIDI standard**: AdLib implements full GM-ish bank of 128 instruments + 128 fallbacks/specials

**Idiomatic to this era, different from modern audio:**
- Modern engines use **wavetable synthesis**, **sample playback**, or **procedural generation**
- This engine uses **static pre-computed FM tables**—reflects the CPU/memory tradeoffs of 1995
- No runtime parameter tuning; patches are fixed at compile-time

**Entries 128–255 reveal design intent:**
- Indices 0–127: Unique, varied timbre data (instrument bank)
- Indices 128–160: Identical repeated entry (`{ { 16, 17 }, { 68, 0 }, ... }` repeated 33 times)
- Indices 161–255: Varied transpose values but reused operator params
- **Likely scenario**: 128–160 are percussion/drum defaults; 161+ are transposed variants or placeholder stubs

## Potential Issues

1. **Incomplete patch bank**: Entries 128–160 repeat identically and entries 161–255 reuse params with only transpose variation. May indicate:
   - Placeholders never filled in during development
   - Intentional fallback defaults for undefined program numbers
   - Unfinished percussion bank

2. **No bounds checking implied**: Code that indexes this array must validate MIDI program numbers 0–255; out-of-range access is undefined. Not evident from this file alone but a typical DOS-era risk.

3. **No versioning/metadata**: If timbres need updating post-release, the entire binary must be rebuilt—no runtime patch loading system visible in this file.
