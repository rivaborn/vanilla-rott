# audiolib/source/gusmidi2.h — Enhanced Analysis

## Architectural Role

This header defines the public interface for the Gravis Ultrasound (GUS) MIDI synthesizer driver, one of several parallel sound card backends in the audio library architecture. The GUS driver sits at the hardware abstraction layer, receiving MIDI commands from higher-level code and translating them into GUS-specific voice allocation and patch management operations. It operates alongside similar backends (AWE32, AdLib FM, generic AL_* MIDI layer) in a modular synthesizer architecture designed to support multiple sound cards of the DOS/early-1990s era.

## Key Cross-References

### Incoming (who depends on this file)
- Unknown from provided cross-reference data (would need grep of `#include "gusmidi2.h"` across codebase)
- Likely callers: game engine audio subsystem, cinematic/dialogue playback, SFX manager
- Probable abstraction layer above: `AL_*` generic MIDI dispatch (seen in cross-ref as parallel driver pattern)

### Outgoing (what this file depends on)
- **D32DosMemAlloc** (`audiolib/source/gus.c`, `irq.c`): DOS real-mode memory allocator for GUS DMA buffers
- **Hardware/DOS extender headers** (not visible in cross-ref): GUS firmware interface, DOS extender runtime, patch file loader
- **Implicit global state**: GUS hardware registers, MIDI channel/voice tables, patch cache in DRAM

## Design Patterns & Rationale

**Parallel Driver Architecture**: The codebase implements multiple synthesizer backends (`GUSMIDI_*`, `AWE32_*`, `AL_*`, `ADLIBFX_*`) with identical function signatures (init/shutdown, note on/off, control change, pitch bend). This allows compile-time or runtime selection of the active sound card without changing game logic.

**Error-Code Enum Pattern**: Consistent with `AWE32_ErrorString`, `ADLIBFX_ErrorString` (seen in cross-ref), uses negative codes for errors (`GUS_Warning`, `GUS_Error`) and positive for specific failures. This was the standard error reporting before C99 errno patterns.

**Lazy Patch Loading**: `GUSMIDI_ProgramChange` likely defers `GUSMIDI_LoadPatch` until first use per channel, rather than pre-loading all patches at init (inferable from function separation and GUS DRAM constraints—typical GUS cards had only 512–4 MB).

**DOS Memory Segregation**: `D32DosMemAlloc` allocates from real-mode-addressable memory specifically for GUS DMA operations, separate from extended memory. This reflects the 1990s era where sound cards used DMA and required low memory access.

## Data Flow Through This File

```
Game/Cinematic Code
    ↓
GUSMIDI_Init (initialize GUS hardware and MIDI state)
    ├→ GUS_Init (hardware setup)
    ├→ D32DosMemAlloc (allocate DMA buffers)
    └→ patch cache initialized to empty
    ↓
MIDI Event Loop:
    ├ GUSMIDI_ProgramChange(chan, prog)
    │   └→ GUSMIDI_LoadPatch(prog) [lazy load if needed]
    │       └→ GUS_GetPatchMap(name) [resolve prog# to filename]
    │           └→ Load from disk, write to GUS DRAM
    ├ GUSMIDI_NoteOn(chan, note, vel) → allocate voice, set pitch/vol
    ├ GUSMIDI_NoteOff(chan, note, vel) → release voice
    ├ GUSMIDI_ControlChange(chan, cc, val) → mod voice parameters
    ├ GUSMIDI_PitchBend(chan, lsb, msb) → adjust voice pitch
    └ GUSMIDI_SetVolume(vol) → attenuate all voices
    ↓
GUSMIDI_Shutdown (release patches and voices)
    └→ GUS_Shutdown (hardware reset)
```

## Learning Notes

**1990s Sound Card Design**: The GUS was a programmable synthesizer card with on-board DRAM for sample-based synthesis. Unlike later wavetable cards (AWE32), it required patches to be pre-loaded into limited memory and managed per-voice—explaining the separate load/unload functions and patch map indirection.

**DOS Extender Integration**: `D32DosMemAlloc` reveals how 1990s games bridged 32-bit protected mode with 16-bit real-mode hardware: allocate low memory explicitly, use DMA-safe buffers. Modern engines abstract away this complexity.

**Parallel Backends vs. Unified Layer**: The existence of `AL_*` functions alongside device-specific drivers (`GUSMIDI_*`, `AWE32_*`) suggests a two-tier design: game calls generic AL_* MIDI functions, which dispatch to the active backend at runtime. This header alone doesn't show that dispatch, but the naming pattern is clear from cross-references.

**Stateful Channel Model**: MIDI channels (0–15) are stateful; `GUSMIDI_ProgramChange` and `GUSMIDI_NoteOn` operate on channel contexts. Unlike sequencer-based music systems, this is raw MIDI event handling—likely fed from a MIDI file parser or real-time input elsewhere.

## Potential Issues

**No Polyphony Limits Declared**: Functions like `GUSMIDI_NoteOn` take no voice-allocation parameters. The GUS has a fixed number of voices (typically 32); this header doesn't expose voice count or query remaining voices. Over-allocating would silently steal voices from earlier notes.

**Patch Lifecycle Unclear**: `GUSMIDI_ReleasePatches()` exists but no function to query loaded patches or detect patch memory exhaustion. Loading a program might fail silently if DRAM is full (inferred from error code `GUS_OutOfDRAM`).

**Memory Ownership Ambiguity**: `D32DosMemAlloc` signature takes only size; no corresponding `D32DosMemFree` appears in this header. Memory cleanup is likely internal to `gus.c` and `GUSMIDI_Shutdown`, but not explicit here.

---

*This analysis assumes GUS hardware knowledge and 1990s DOS extender memory models. The cross-reference data confirms this is one of N parallel synthesizer backends, likely invoked through a higher-level dispatch layer not visible in this header alone.*
