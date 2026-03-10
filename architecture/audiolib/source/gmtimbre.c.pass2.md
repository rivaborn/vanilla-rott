# audiolib/source/gmtimbre.c — Enhanced Analysis

## Architectural Role
This file serves as the **instrument timbre database** for the AdLib FM synthesizer subsystem in the audio library. It bridges MIDI program change commands (Program 0–127, plus percussion Programs 128–255) to hardware-level FM operator parameters, enabling the synthesizer to reproduce diverse instrument timbres. The 256-entry bank is directly indexed during voice allocation (`AL_SetVoiceTimbre` in `al_midi.c`), making timbre selection a O(1) lookup operation critical for real-time synthesis.

## Key Cross-References

### Incoming (who depends on this file)
- **`AL_SetVoiceTimbre()`** (`al_midi.c`) — Called when a MIDI Program Change arrives; retrieves a TIMBRE struct from `ADLIB_TimbreBank` and programs the active FM voice with Envelope, Level, Waveform, and Feedback parameters
- **`AL_ProgramChange()`** (`al_midi.c`) — Routes program selections through the timbre bank lookup
- **Synthesizer voice driver** — FM hardware registers are written based on TIMBRE fields (SAVEK, Level, Env1/Env2, Wave, Feedback map to OPL2 register layout)

### Outgoing (what this file depends on)
- **None** — This is a pure data module; no function calls or external state dependencies

## Design Patterns & Rationale

1. **Direct Hardware Encoding:** Fields like `SAVEK[2]`, `Env1[2]`, `Env2[2]` directly encode 8-bit OPL2 FM synthesizer register values. No translation layer—values are bit-packed to match Yamaha OPL2 specification. **Why:** 1990s hardware abstraction was minimal; direct encoding was faster and transparent to audio developers familiar with FM synthesis.

2. **Two-Operator FM Model:** Each TIMBRE represents a two-operator algorithm (Operator 0 and Operator 1 with feedback). **Why:** OPL2/OPL3 chips are limited to carrier–modulator topologies; this struct mirrors the hardware constraint.

3. **MIDI Metadata Alongside Synthesis:** Transpose (signed 8-bit) and Velocity (signed 8-bit) sit beside synthesis parameters. **Why:** Allows per-instrument pitch shifting and dynamic voice velocity scaling without a separate mapping table.

4. **Fixed 256-Entry Bank:** Indices 0–127 (General MIDI), 128–255 (Percussion/Extended). **Why:** MIDI standard; ceiling on program numbers; fast table indexing.

5. **Duplication in Percussion Range (128–255):** Many entries (e.g., rows 128–191) are repeated or sparse with zeros. **Why:** Likely placeholder data or fallback entries; drum/percussion synthesis may reuse a few tuned timbres rather than needing 128 unique percussion instruments.

## Data Flow Through This File

```
MIDI ProgramChange(N)
  ↓
AL_ProgramChange() → AL_SetVoiceTimbre(voice, N)
  ↓
ADLIB_TimbreBank[N] lookup → TIMBRE struct
  ↓
Extract fields: SAVEK, Level, Env1, Env2, Wave, Feedback, Transpose, Velocity
  ↓
Write to OPL2 hardware registers for active FM voice
  ↓
Synthesizer outputs audio for selected instrument
```

**Key state transition:** Static data → (indexed at runtime) → ephemeral voice configuration (no persistent state in this file itself).

## Learning Notes

- **FM Synthesis Era Pattern:** This file encodes the 1990s practice of hand-tuning FM operator parameters (complex and manual before graphical editors). Each row is likely a precious, hard-won instrument definition created by audio engineers.
- **OPL2 Constraints:** The `Feedback` field (single byte) and two-operator limit reflect OPL2's hardware design; modern synthesizers offer more flexible operator stacking.
- **No Runtime Reconfiguration:** This bank is read-only at runtime. There's no mechanism to hot-load or override timbres dynamically (unlike modern wavetable synths).
- **MIDI-to-Hardware Bridge:** This file exemplifies how MIDI (a software protocol) was directly mapped to FM hardware registers in retro engines—no intermediate sound font or preset system.

## Potential Issues

1. **Incomplete Percussion Data (rows 128–255):** Heavy repetition suggests the percussion bank was either left incomplete or reuses a single "fallback" timbre for untuned entries. If these entries are actually selected during gameplay, audio may be sparse or disappointing.
2. **No Bounds Validation Here:** The code assumes callers validate MIDI program numbers before indexing. Passing an out-of-range value would cause undefined behavior (though likely wrapped by calling code).
3. **Hardcoded Transpose Values:** The Transpose field is baked in per instrument. If a piece needs transposition, it's applied at the voice playback level, not here—this is correct but worth noting for audio porting.
