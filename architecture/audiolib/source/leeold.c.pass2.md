# audiolib/source/leeold.c — Enhanced Analysis

## Architectural Role

This file serves as the **core timbre resource library** for the AdLib FM synthesis subsystem. It bridges MIDI-level sound requests (from `al_midi.c`) to hardware-level FM operator configurations, enabling both melodic instrument playback and percussion effects. The "leeold" naming suggests this is a legacy/fallback instrument bank (possibly predating a newer synthesizer or supporting backward compatibility with older ROTT versions).

## Key Cross-References

### Incoming (who depends on this file)

- **`audiolib/source/al_midi.c` → `AL_RegisterTimbreBank()`**: Almost certainly called at init time with `ADLIB_TimbreBank` as the global instrument table. This registers the 256 timbres for MIDI program-change operations.
- **`audiolib/source/adlibfx.c` → Sound effect playback**: Likely indexes into `ADLIB_TimbreBank` when triggering FM-synthesized SFX (via voice allocation functions like `AL_SetVoiceTimbre`).
- **Game audio subsystem (via `AL_ProgramChange`)**: When a MIDI track or SFX requests instrument N (0–255), the engine looks up `ADLIB_TimbreBank[N]` and configures the FM operator pair accordingly.

### Outgoing (what this file depends on)

- **None**: Pure data. No function calls, no external includes. Defines only static structures consumed by the audio library.

## Design Patterns & Rationale

**Static Lookup Table Pattern**: Pre-computed FM synthesis parameters eliminate runtime synthesis overhead. Each `TIMBRE` entry encodes 13 hardware values (2×SAVEK, 2×Env, 2×Wave, 1×Feedback) in a compact 7-byte structure, supporting ~256 distinct sounds.

**MIDI Mapping Convention**: 
- `ADLIB_TimbreBank[0–255]` directly corresponds to MIDI program numbers.
- `PercussionTable[0–127]` maps MIDI percussion notes (GM standard) to timbre indices + pitch offsets.

**Sparse Percussion Table**: Most percussion entries (0–34, 80–128) map to timbre 128 (disabled), concentrating actual drum timbres in ranges ~35–79 and ~109–211. This suggests selective percussion support (likely kicks, snares, cymbals, etc.) while avoiding unnecessary drum kit expansion.

**Operator Pair Design**: Each `TIMBRE` struct follows the Yamaha OPL FM model (modulator on left, carrier on right), with attack/decay/sustain/release packed as bitfields in `SAVEK`, `Env1`/`Env2` bytes.

## Data Flow Through This File

1. **Initialization**: Engine startup → `AL_Init()` / `AL_RegisterTimbreBank()` → copies/caches `ADLIB_TimbreBank` into MIDI subsystem state
2. **MIDI Playback**: `AL_ProgramChange(channel, program)` → indexes `ADLIB_TimbreBank[program]` → writes FM operator registers to hardware
3. **Percussion**: `AL_NoteOn(channel, percussion_note, velocity)` → looks up `PercussionTable[percussion_note]` → selects timbre + pitch → plays sound
4. **SFX Playback**: Sound effect trigger → allocate voice → `AL_SetVoiceTimbre(voice, timbre_id)` → configure FM operators from `ADLIB_TimbreBank[timbre_id]`

## Learning Notes

**Era-Specific Design**: This exemplifies 1990s game audio architecture:
- Hardware synthesizers (AdLib/OPL2/OPL3) required pre-programmed patch banks, not real-time sound design.
- FM synthesis was the dominant 8-bit/16-bit sound standard, replacing sample-based audio until CD-ROM era.
- Static timbre tables were the only practical approach (no dynamic synthesis, no sample streaming).

**Modern Contrast**: Current engines (Unreal, Unity) use:
- Dynamic synthesizers or pre-recorded sample banks (memory abundant).
- Runtime synthesis (GPU-accelerated or CPU streaming).
- No direct hardware register programming (OS abstraction layer).

**FM Synthesis Deep Dive**: The TIMBRE structure encodes classic Yamaha operator pairs. Understanding these numbers requires knowledge of OPL register layout (attack rates, release rates, waveform selection, modulation index). This file is a **lookup table for FM synthesis experts only**.

## Potential Issues

1. **Index Out of Bounds**: If MIDI code ever requests `ADLIB_TimbreBank[N]` where N > 255 or `PercussionTable[N]` where N > 127 without bounds checks, undefined behavior occurs. The globals are global-scoped and non-const, so no compile-time safety.

2. **Silent Percussion**: ~110 out of 128 percussion entries are disabled (timbre 128, key 0). If a game tries to play unmapped percussion (e.g., MIDI note 50), it will silently fail or produce no sound. No error or warning mechanism visible here.

3. **No Version Tracking**: The file name `leeold.c` suggests this is legacy data. If newer timbre banks were added elsewhere (e.g., `leeadlib.c`, `adlib_hq.c`), there's no mechanism to select between them at runtime—hard-coded usage throughout.
