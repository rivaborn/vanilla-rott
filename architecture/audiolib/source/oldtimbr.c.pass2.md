# audiolib/source/oldtimbr.c — Enhanced Analysis

## Architectural Role
This file is the FM synthesis preset library for the audio subsystem's OPL (AdLib) MIDI layer. It serves as the immutable sound palette—every melodic instrument and percussion sound playable through the MIDI interface is defined here. At initialization, `AL_RegisterTimbreBank()` (from `al_midi.c`) loads `ADLIB_TimbreBank` into the synthesizer; at runtime, MIDI events (`AL_NoteOn`, `AL_ProgramChange`) index this table to configure OPL operators.

## Key Cross-References

### Incoming (who depends on this)
- **`al_midi.c`**: `AL_RegisterTimbreBank()` and `AL_SetVoiceTimbre()` consume `ADLIB_TimbreBank` to configure voice FM parameters
- **`al_midi.c`** (MIDI event handlers): `AL_NoteOn()`, `AL_ControlChange()`, `AL_ProgramChange()` indirectly reference timbres via voice allocation
- **Percussion routing**: MIDI percussion events (channels 9/10) are routed via `PercussionTable` to select drum timbres rather than melodic ones

### Outgoing (what this depends on)
- None. Pure data; no includes, no external symbols referenced.

## Design Patterns & Rationale

**ROM-table synthesis preset model**: All 174 presets are hard-coded at compile time. This is a 1990s design—no dynamic loading, no serialization, no editor UIs. Every soundfont is baked into the binary.

**Two-operator FM synthesis**: Each `TIMBRE` entry encodes parameters for two OPL operators (carrier + modulator). The raw register values (0–255) in `SAVEK`, `Level`, `Env1`, `Env2`, `Wave` map directly to OPL2 chip registers. This reflects OPL's low-level, register-oriented API, not a high-level synthesizer abstraction.

**Sparse percussion mapping**: `PercussionTable[128]` maps all 128 MIDI drum notes, but indices 0–35 and 88–127 default to timbre 128 (a silent/invalid entry, likely used as a guard). Only indices 36–88 (the standard GM drum kit range) use varied timbres (129–173). This is tight MIDI drum standard compliance but with minimal redundancy.

## Data Flow Through This File

```
Initialization:
  AL_Init() → AL_RegisterTimbreBank(ADLIB_TimbreBank, 174)
             → copies timbres into chip state

At runtime (melodic):
  MIDI ProgramChange (evt) → AL_ProgramChange()
                           → AL_SetVoiceTimbre(voice, program)
                           → reads ADLIB_TimbreBank[program]
                           → writes operator params to OPL chip

At runtime (percussion):
  MIDI NoteOn on channel 9/10 → AL_NoteOn()
                              → PercussionTable[note] lookup
                              → read timbre ID + key
                              → AL_SetVoiceTimbre(voice, timbre_id)
                              → reads ADLIB_TimbreBank[timbre_id]
                              → writes operator params to OPL chip
```

The `DRUM_MAP.Key` field is a repitched note—drums usually ignore MIDI velocity as pitch, but some drums (timpani, tuned percussion) can be retuned via this field.

## Learning Notes

**FM synthesis register encoding**: Modern sound designers use spectral/wavetable editors; here, parameters are opaque OPL register dumps. A developer studying this file learns how Yamaha OPL FM synthesis actually worked—operator connections, ADSR envelopes, waveform selection—at the silicon level. This is hardware synthesis, not software synthesis.

**Era-specific pragmatism**: The "oldtimbr.c" name suggests this predates a newer timbre format (possibly `newtimbr.c` or MIDI synthesizer retooling). Hard-coded arrays were practical when memory and binary size were critical constraints. No compression, no metadata, just raw 22-byte chunks per timbre.

**MIDI standard compliance with constraints**: The percussion table respects the General MIDI drum kit (GM, indices 35–81 = acoustic bass drum to open triangle), but the broad 128-entry table and dummy entries (timbre 128) hint at OPL's limitations—not every synth can play every drum simultaneously or with full expression.

## Potential Issues

- **Undocumented parameter semantics**: TIMBRE fields (e.g., `Level[2]`, `Wave[2]`) are raw register values with no semantic documentation. Modifying them requires OPL datasheet reverse-engineering.
- **Silent default (timbre 128)**: Percussion indices 0–35 and 88–127 map to a non-existent timbre (index 128 > 174), risking index-out-of-bounds if not guarded in `AL_SetVoiceTimbre()`. Likely intentional (silencing off-spec notes), but relies on bounds-checking elsewhere.
- **No version/checksum**: If this timbre bank ever needs to be hot-swapped or patched, there's no versioning or CRC to detect corruption.
