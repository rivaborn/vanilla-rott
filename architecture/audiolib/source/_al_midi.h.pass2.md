Looking at the cross-reference data provided, I can now write an enriched second-pass analysis:

---

# audiolib/source/_al_midi.h — Enhanced Analysis

## Architectural Role
This is a **private internal header** (underscore prefix convention) defining static helper functions and data structures for the AdLib FM synthesizer backend. It sits between the public MIDI interface (`al_midi.h`) and the implementation (`al_midi.c`), encapsulating low-level voice and channel management. The file serves as the foundation for the audio library's hardware synthesis layer, complementing higher-level sound card backends (AWE32, GUS, Blaster).

## Key Cross-References

### Incoming (Dependencies on this file)
- **`al_midi.c`** — Includes this header and implements all declared static functions
- **Public MIDI interface** (`al_midi.h`) — Exposed functions like `AL_NoteOn`, `AL_NoteOff`, `AL_Init` depend on the voice/channel management functions declared here
- **Higher-level audio system** — Via `al_midi.c`, the subsystem is integrated into broader audio playback (Blaster, AWE32, AdLibFX modules handle effects/mixing)

### Outgoing (Dependencies this file declares)
- **`ADLIB_TimbreBank[256]`** — Global extern array of instrument definitions; loaded via `AL_RegisterTimbreBank` and indexed by MIDI program numbers
- **Hardware registers** (constants `alFreqH=0xb0`, `alEffects=0xbd`) — Actual FM synthesis happens via register writes inside the implementation functions
- **No C library dependencies** — Pure hardware abstraction, no stdlib calls in the interface

## Design Patterns & Rationale

**Doubly-Linked Voice List Pattern**
- `VOICELIST` (start/end pointers) manages active voices with `VOICE` nodes
- Rationale: Supports O(1) voice allocation/deallocation, typical for real-time audio synthesis where voices are frequently added/removed during note events

**Per-Channel State Multiplexing**
- `CHANNEL` struct holds timbre, pitch bend, detune, volume, and a linked list of active voices
- Each MIDI channel (0–15) gets one `CHANNEL` struct; voices are allocated per note on that channel
- Rationale: Separates channel-wide settings (volume, pan) from per-voice pitch/envelope, matching MIDI semantics

**Timbre Bank Abstraction**
- `TIMBRE` stores 8 bytes of FM operator parameters (SAVEK, envelopes, waveform, feedback, transpose, velocity curve)
- Pre-loaded into global `ADLIB_TimbreBank[256]`, indexed by MIDI program number
- Rationale: Pre-computed timbres avoid real-time synthesis of instrument definitions; common pattern in DOS-era synthesizers

**Hardware Constant Layers**
- Enums (`cromatic_scale`, `octaves`) provide symbolic note and octave offsets, ORed together to form F-Number register values
- Rationale: Abstracts AdLib's 10-bit frequency encoding (octave + note combo), making pitch math more readable

## Data Flow Through This File

1. **Initialization**: `AL_ResetVoices()` clears voice state; timbre bank populated via `AL_RegisterTimbreBank()`
2. **Note On**: `AL_GetVoice()` allocates a voice slot → `AL_SetVoiceTimbre()` loads instrument → `AL_SetVoicePitch()` writes F-Number
3. **Channel Updates**: `AL_SetChannelVolume/Pan/Detune()` modify `CHANNEL` state; `AL_CalcPitchInfo()` pre-computes pitch lookup tables
4. **Sustain**: Per-frame updates via pitch bend (modifies `pitchleft`/`pitchright` fields), triggering `AL_SetVoicePitch()` register writes
5. **Note Off**: Voice deallocated (function not declared in this header; likely in `al_midi.c`)

## Learning Notes

**Idiomatic to Early '90s Game Audio**
- Tight hardware coupling: no abstraction between logical voices and physical chip slots (18 slots, 9 polyphonic voices)
- Per-channel detune (stereo effect on AdLib stereo variants) handled via `pitchleft`/`pitchright` split
- Velocity curve baked into timbre (`TIMBRE.Velocity` field), not per-voice

**Modern Engines Do This Differently**
- Today's engines separate synthesis from voice management (voices → schedulers → synthesis engines)
- Timbre data would be asset-loaded, not global arrays
- Hardware abstraction is deeper; FM synthesis is a renderer plugin, not the core

**Design Insight: The Voice-Channel Asymmetry**
- Unlike MIDI, which is inherently per-channel, this implementation ties voices to specific hardware slots
- `AL_GetVoice(channel, key)` finds or allocates a voice, potentially reusing old voices if 18-slot limit is hit
- This explains the `AL_VoiceNotFound` error code: true voice starvation is possible

## Potential Issues

- **No voice stealing algorithm visible**: If all 18 hardware slots are occupied, `AL_AllocVoice()` likely steals the oldest voice, but the cutoff logic is in `al_midi.c` (not visible here)
- **Timbre bank size fixed at 256**: No dynamic timbre allocation; assumes MIDI programs stay within 0–255
- **No overflow protection on pitch bend**: `pitchleft`/`pitchright` fields suggest pitch bend can shift frequency left/right, but max range is not constrained in this header

---

*Note: Architecture context was truncated in the provided materials; cross-references limited to function definitions. Full incoming dependency graph (game loop, MIDI parser, etc.) would require analysis of `al_midi.h` and callers.*
