# audiolib/source/al_midi.h — Enhanced Analysis

## Architectural Role
This file defines the public API for Adlib/OPL2 FM synthesizer MIDI support in a pluggable audio subsystem. It is one of several parallel synthesizer interfaces (alongside `AWE32` and `BLASTER` visible in the cross-reference), suggesting the engine could target different hardware at compile/runtime. The file bridges game logic and low-level hardware voice/register management, establishing the contract that higher-level modules (like `ADLIBFX`, which wraps these functions for effects) depend on.

## Key Cross-References

### Incoming (who depends on this file)
- **Game code** in `rott/` directory calls these MIDI functions for music/sound playback
- **ADLIBFX subsystem** (`audiolib/source/adlibfx.c/h`) wraps these functions for effects and sound object management
- Likely **cinematic/sequencer code** uses `AL_NoteOn`, `AL_ControlChange` during scripted sequences

### Outgoing (what this file depends on)
- **Implementation**: `audiolib/source/al_midi.c` implements all declared functions
- **Internal helpers**: `audiolib/source/_al_midi.h` (private header) provides lower-level voice management (`AL_AllocVoice`, `AL_GetVoice`, `AL_SetVoiceVolume`, `AL_SetVoiceTimbre`, `AL_ResetVoices`, `AL_CalcPitchInfo`)
- **Hardware**: Direct I/O to Adlib port `0x388` via `AL_SendOutput` / `AL_SendOutputToPort`

## Design Patterns & Rationale
**Hardware Abstraction Layer (HAL)**: Multiple synthesizer APIs (AL_*, AWE32_*, BLASTER_*) with identical logical signatures (`Init`, `NoteOn`, `NoteOff`, `ProgramChange`, `ControlChange`, `Shutdown`) suggest a swappable card interface. The game likely selects at runtime which hardware API to use.

**Resource Pool Pattern**: Voice reservation/release (`AL_ReserveVoice`, `AL_ReleaseVoice`) and internal voice allocation (`AL_AllocVoice` in `_al_midi.h`) implement manual voice pooling—necessary on hardware with fixed polyphony (OPL2 has ~9 operators total).

**Hardware Initialization Sequence**: `AL_DetectFM` → `AL_Init` → `AL_RegisterTimbreBank` → playback. This is procedural and order-dependent, reflecting bare-metal hardware constraints.

## Data Flow Through This File
1. **Initialization**: Game calls `AL_DetectFM()` (probe), then `AL_Init(soundcard)` (configure), then `AL_RegisterTimbreBank(timbres)` (load instruments).
2. **Playback**: `AL_NoteOn(channel, key, vel)` → internally allocates voice → `AL_SendOutput()` writes to hardware registers. `AL_ControlChange()` / `AL_SetPitchBend()` modulate live notes.
3. **Cleanup**: `AL_Reset()` and `AL_Shutdown()` release hardware.

Voice lifecycle is opaque to caller—all voices are allocated on-demand by `AL_NoteOn` (via internal `AL_AllocVoice` in `_al_midi.h`).

## Learning Notes
- **Era-specific**: Adlib/OPL2 is 1980s FM synthesis requiring manual voice allocation and per-note operator configuration (timbre bank). Modern engines use software synthesis or defer to audio middleware.
- **MIDI Compliance**: 14-bit pitch bend (LSB/MSB split), 7-bit volume/CC values, 128 programs per channel—standard MIDI constraints mapped to limited hardware.
- **Volume Constants**: `AL_MaxVolume = 127` and `AL_DefaultChannelVolume = 90` suggest normalized MIDI volume compressed into hardware's smaller dynamic range.
- **Internal vs. Public**: The `_al_midi.h` file (voice helpers, pitch calculation, timbre setting) is _not_ in this public header, indicating encapsulation of implementation details. Game code is _not_ expected to manage voices directly.

## Potential Issues
- **Hard-coded hardware port** (`0x388`): No abstraction for port selection; assumes single Adlib card. Multiple cards not supported.
- **No error handling in most functions**: Many return `void` despite potential hardware failures; only `AL_Init`, `AL_DetectFM`, voice functions return error codes.
- **Manual pitch bend split (LSB/MSB)**: `AL_SetPitchBend(channel, lsb, msb)` requires caller to decompose 14-bit value; higher-level API could hide this.
