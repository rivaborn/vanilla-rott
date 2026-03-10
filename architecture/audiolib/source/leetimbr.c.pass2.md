# audiolib/source/leetimbr.c — Enhanced Analysis

## Architectural Role
This file provides the **global FM instrument bank** for the AdLib FM synthesis subsystem. It sits between the MIDI layer (`al_midi.c`) and real-time FM synthesis (`adlibfx.c`), storing 256 pre-configured OPL2 operator parameter sets that are indexed via MIDI Program Change messages. When a voice needs a timbre, `AL_SetVoiceTimbre` (from `al_midi.c`) fetches an entry from `ADLIB_TimbreBank` and programs the hardware accordingly.

## Key Cross-References

### Incoming (who depends on this file)
- **`al_midi.c:AL_SetVoiceTimbre`** — reads `ADLIB_TimbreBank[index]` to apply timbre config to an active synthesizer voice
- **`al_midi.c:AL_RegisterTimbreBank`** — accepts a pointer to a TIMBRE array (allows runtime timbre bank swapping, though this file provides the default)
- **`adlibfx.c` audio synthesis loop** — references timbre settings during FM synthesis voice generation (indirectly via voice state)

### Outgoing (what this file depends on)
- None; this is self-contained data with no external dependencies

## Design Patterns & Rationale

**Static ROM-based Instrument Table**
- All 256 instruments are hardcoded, matching the Yamaha OPL2 hardware design paradigm of the era (pre-synthesis FM chips, not sample-based)
- Reflects hardware constraints: OPL2 has only 9 polyphonic voices and 256 timbre slots, so the engine caches all possible sounds at startup

**Template/Default Pattern**
- ~40% of entries repeat identical configurations (e.g., indices 2–10, 22–24, 50–52)
- Suggests a **fallback/placeholder strategy**: many MIDI program numbers map to the same synthesizer preset, reducing tuning burden
- Single heavily-used default config (33, 17, 17, 0, 163, 196, 67, 34, 2, 0, 13, 0) appears ~60 times

**Drum Kit Variant (indices ~190–256)**
- Non-zero Transpose and Velocity fields in later entries align with **General MIDI drum kit layout** (GM channels 9 in MIDI map to fixed percussion)
- Transpose values (36, 48, 52, etc.) correspond to pitch offset for multi-hit drum samples tuned to specific note ranges
- Examples: kick drum at index ~242 has Transpose=0, Velocity=36; snare at ~248 has Transpose=0, Velocity=48

**2-Operator Feedback FM Topology**
- Each TIMBRE encodes two Yamaha OPL2 operators (modulator/carrier) with independent envelopes
- Feedback field (0–15) enables **self-modulation** on the modulator, core to classic FM bell/electric piano tones

## Data Flow Through This File

1. **Initialization**: Engine boots, `ADLIB_TimbreBank` is globally resident in binary image
2. **Selection**: MIDI event (Program Change) specifies program number [0–255] → index into `ADLIB_TimbreBank`
3. **Application**: `AL_SetVoiceTimbre(voice, program)` reads `ADLIB_TimbreBank[program]`, extracts SAVEK/Level/Env1/Env2/Wave/Feedback values
4. **Synthesis**: Real-time FM voice loop applies Transpose and Velocity scaling, writes OPL2 registers with envelope and waveform params
5. **Output**: Synthesized carrier wave sent to DSP/mixer

## Learning Notes

**Idiomatic to 1990s FM Hardware Synthesis**
- This hardcoded approach contrasts sharply with modern sample-based engines (e.g., General MIDI devices that load SoundFont banks at runtime)
- FM tuning was **domain expertise**: each entry represents careful hand-tuning by an audio engineer (likely "Lee") to achieve recognizable timbres on OPL2's limited topology
- No parameter interpolation or morphing—instruments are static snapshots

**Yamaha OPL2 Specifics Embedded Here**
- The SAVEK, Env1, Env2 field widths (0–255) map directly to OPL2 register bit layouts (operator Key Scale Level, ADSR rates)
- Wave field (0–3) selects sine/half-sine/abs-sine/pulse-sine, a hardware constraint; modern FM would allow arbitrary harmonic shaping
- Feedback (0–15) is OPL2's only modulation routing option; complex FM chains require external sequencing

**Multitimbrality Trade-off**
- The 256-slot design enables **polyphonic timbre variety** within 9 hardware voices, but at the cost of **no runtime timbral morphing** (each voice either uses preset A or B, not A→B transition)
- Compare to modern plugin FM: `ADSR_NormalizerState` and automated parameter curves would replace these static tuples

## Potential Issues

**Hardcoded Bank Inflexibility**
- New instruments require recompilation; end-users cannot add or modify sounds
- Bank swapping via `AL_RegisterTimbreBank` suggests the engine *intended* multi-bank support, but only one bank is ever compiled in

**Preset Repetition Risk**
- Heavy reuse of the same default config (60+ slots) means a single tuning error affects many instruments
- No versioning or checksums; if binary is corrupted, no runtime validation detects it

**Drum Kit Mapping Ambiguity**
- Transpose and Velocity are repurposed for percussion but the semantics aren't formalized in the struct comment; coupling is implicit
- If someone naively remaps a drum entry, the voice will synthesize at wrong pitch/amplitude without obvious cause

---

**Summary**: This file is a **canonical FM instrument ROM** for AdLib synthesis—a static, hand-tuned collection reflecting both the musical expertise of its author and the hardware constraints of early 1990s OPL2 synthesis. Its architecture is typical of the era but would be replaced by sample-based wavetable engines or soft synths in modern ports.
