# audiolib/source/al_midi.c — Enhanced Analysis

## Architectural Role

This file forms the **MIDI-to-FM synthesis driver** within a multi-backend audio abstraction layer. It bridges high-level MIDI events (from game or sequence) to OPL2/OPL3 FM hardware, operating as one of several parallel hardware drivers (alongside AWE32, GUS, Sound Blaster DSP). The voice pool management allows coexistence with `adlibfx.c` (SFX driver) via the `AL_ReserveVoice()` API, enabling synchronized MIDI and sound-effect playback on shared FM hardware.

## Key Cross-References

### Incoming (Who Depends on This File)
- **Primary MIDI clients** (not visible in cross-ref excerpt, but inferred): Game engine MIDI sequencer and real-time MIDI input handlers route note/controller events here via `AL_NoteOn()`, `AL_NoteOff()`, `AL_ControlChange()`, `AL_SetPitchBend()`, `AL_ProgramChange()`.
- **adlibfx.c** (SFX synthesizer): Shares FM voice pool via `AL_ReserveVoice()` / `AL_ReleaseVoice()` to claim up to 9 voices for sound effects without colliding with MIDI notes.
- **Initialization chain**: Called by audio subsystem boot via `AL_Init()` (likely from a master audio initialization in `al_main.c` or similar, not shown in excerpt).

### Outgoing (What This File Depends On)
- **Hardware I/O abstraction**: `outp()` / `inp()` (from `conio.h`) → Adlib register writes with timing-critical delays.
- **Card detection & configuration**: `BLASTER_GetEnv()`, `BLASTER_GetCardSettings()` (from `blaster.h`) → Sound Blaster environment parsing to locate hardware ports.
- **Memory locking**: `DPMI_LockMemoryRegion()`, `DPMI_Lock()` / `DPMI_Unlock()` (from `dpmi.h`) → Locks entire functions in low-memory DOS real-mode for interrupt-safe operation (marked `AL_LockStart` → `AL_LockEnd`).
- **Interrupt safety**: `DisableInterrupts()`, `RestoreInterrupts()` (from `interrup.h`) → Protects voice pool modifications during voice reservation.
- **Data structures**: `LL_Remove()`, `LL_AddToTail()` (from `ll_man.h`) → Linked-list operations for voice pool management.
- **User parameters**: `USER_CheckParameter()` (from `user.h`) → Checks runtime flags (e.g., `NO_ADLIB_DETECTION`).
- **Timbre bank**: `ADLIB_TimbreBank[]` (defined elsewhere, likely `adlib.c` or header) → Global FM instrument definitions (256 × 13-byte packed structures).
- **Hardware constant**: `ADLIB_PORT` (defined elsewhere) → Primary Adlib port address.

## Design Patterns & Rationale

**Pattern: Hardware Abstraction Layer (HAL)**
- Multiple parallel drivers (AL_*, AWE32_*, GUS_*) implement identical MIDI interfaces for different hardware.
- Allows game to remain hardware-agnostic; driver selection at init time.

**Pattern: Voice Pool & Lazy Allocation**
- Voices are preallocated as a linked list (`Voice_Pool`) on init, not dynamically allocated per note.
- Rationale: Avoids malloc latency during real-time note-on; deterministic timing.
- Reserved voices API allows SFX driver to claim contiguous slots without dynamic negotiation.

**Pattern: Register Write Buffering via Timbre Cache**
- `Voice[].timbre` caches current patch; skips reprogramming if unchanged (`AL_SetVoiceTimbre()` early return).
- Rationale: FM register writes are slow; avoid redundant ops on same timbre.

**Pattern: Stereo Simulation via Voice Doubling**
- Stereo mode uses two voices per note (one per OPL3 chip/port) with separate pan/detune.
- Rationale: Classic FM technique; emulates wide stereo on hardware lacking true stereo support.

**Pattern: Fine-Tuning Lookup Table**
- Static `NotePitch[FINETUNE_MAX+1][12]` precomputed at compile-time instead of runtime calculation.
- Rationale: Period-accurate DOS code; avoids floating-point; comment hints original code was slower/dynamic.

**Pattern: Interrupt-Locked Core Functions**
- Functions from `AL_SendOutputToPort()` to `AL_LockEnd()` are memory-locked for interrupt-safe operation.
- Rationale: Allows safe voice allocation/deallocation from async interrupt context (e.g., MIDI in from serial port).

## Data Flow Through This File

```
MIDI Input (NoteOn/Off/CC/PitchBend)
  ↓
AL_NoteOn/Off/ControlChange/SetPitchBend
  ↓
Voice Allocation (AL_AllocVoice from Voice_Pool)
  ↓
Voice State Update (Voice[].key, .velocity, .port, .channel)
  ↓
Timbre Lookup (Channel[ch].Timbre or patch+128 for drums)
  ↓
AL_SetVoiceTimbre → Program FM operators via AL_SendOutputToPort
  ↓
AL_SetVoiceVolume → Calculate velocity×volume×pan, write amplitude regs
  ↓
AL_SetVoicePitch → Lookup NotePitch[finetune][note%12], add octave, write freq/octave regs
  ↓
FM Hardware Synthesis (Adlib card renders audio)
```

**State Transitions:**
- `Voice.status`: NOT_ALLOCATED → NOTE_ON → NOTE_OFF → NOT_ALLOCATED
- `Channel.PitchBend`: Triggered by RPN LSB/MSB, reprograms all active voices on channel via `AL_SetVoicePitch()` loop.

## Learning Notes

**Idiomatic DOS-Era FM Synthesis Patterns:**
1. **Fixed voice count**: 9 voices (vs. modern engines' unlimited polyphony). Reflects OPL2 hardware limit.
2. **Operator pair hierarchy**: Each voice maps to two FM operators (carrier + modulator). Indexes via `slotVoice[]` lookup to avoid scatter.
3. **Envelope control via bit fields**: `timbre->Env1/Env2` are packed ADSR values written directly to hardware registers (0x60, 0x80). No envelope generator software; hardware-native.
4. **Volume via attenuation**: FM volume is *inverted*: `volume = (max - velocity * timbre)`. See `AL_SetVoiceVolume()` XOR with 63, `VoiceLevel[]` cache.
5. **Pitch bend as semitone offset**: `AL_SetPitchBend()` converts 14-bit MIDI bend value to semitone increments on `NotePitch` table, not Hz-based pitch.

**Compared to Modern Engines:**
- Modern engines: Voice stealing (round-robin or LRU), unlimited polyphony via soft synths, real-time wavetable morphing.
- ROTT/Adlib: Hard-wired 9 voices, no stealing (just fails allocation), static timbres (loaded once at init).

**Connections to Game Engine Concepts:**
- **Voice pool**: Similar to object pools in entity systems; avoids GC latency.
- **MIDI routing per channel**: Mirrors modern DAW mixer architecture (channel strip with timbre/volume/pan state).
- **Pitch bend via lookup table**: Hardware era equivalent of efficient integer math; avoids floating-point on 386.

## Potential Issues

1. **No voice stealing**: `AL_AllocVoice()` returns `AL_VoiceNotFound` if pool empty. High-frequency MIDI input can cause silent note drops if polyphony exceeds 9. Likely relies on game logic to avoid over-polyphony.

2. **Lazy timbre reprogram**: Changing timbre mid-note (program change on active channel) doesn't retroactively reprogram active voices. Next note gets new timbre. Could cause unexpected timbral glitches if game changes instrument mid-phrase.

3. **Timing dependency on CPU speed**: Delay loops (`for(delay=27; delay>0; delay--)`) are cycle-counted for specific CPU speed. WSL2/emulation environment could cause register write timing violations. Hardcoded `27` and `6` suggest tuning via trial-and-error.

4. **Stereo detune calculation**: `AL_SetVoicePitch()` applies fixed right-channel detune offset. Could sound unnatural if octave changes; no octave compensation logic visible.

5. **Channel 9 (drums) hardcoded**: Assumes MIDI channel 9 is always drums (patch = key + 128). Non-standard MIDI setups would break.

6. **No bounds checking on patch/key**: `ADLIB_TimbreBank[patch]` and `NotePitch[finetune][note % 12]` assume valid indices. Out-of-range MIDI could read uninitialized memory.

---

**Note:** The architecture context failed to load, so detailed subsystem diagrams unavailable. Analysis inferred from cross-reference data and first-pass observations.
