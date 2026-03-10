# rott/sndcards.h — Enhanced Analysis

## Architectural Role

This header establishes the **sound card abstraction layer** at the audio subsystem's boundary. It provides the enumeration that bridges hardware detection to driver selection, enabling the engine to support multiple audio hardware families (FM synthesis via Adlib, wavetable via SoundBlaster/Awe32, external MIDI devices, etc.). The `soundcardnames` enum acts as a dispatch key throughout the audio initialization pipeline.

## Key Cross-References

### Incoming (who depends on this file)
- **Audio initialization code** (likely in `rott/` or `audiolib/source/`) must `#include` this header to reference `soundcardnames` enum values in switch/if statements during card detection and driver selection
- **Configuration/menu code** (e.g., sound setup menus) likely reads or displays this enum to let users select their audio hardware
- The enum value is typically stored in config or state structures to persist user's audio device choice across sessions

### Outgoing (what this file depends on)
- **No explicit dependencies** — this is a pure definitions-only header with no includes
- The corresponding driver implementations (`audiolib/source/adlibfx.h`, `audiolib/source/blaster.h`, `audiolib/source/awe32.h`, `audiolib/source/al_midi.h`, etc. per the cross-reference index) are selected *based on* values from this enum, but this file has no direct include relationships

## Design Patterns & Rationale

**Driver Selection Pattern**: The enum enables a **strategy pattern** or **factory pattern** where each `soundcardnames` value triggers initialization of the corresponding audio driver.

**Rationale for structure**:
- Centralized list prevents driver selection code from hardcoding string names or scattered magic numbers
- `NumSoundCards` sentinel enables array sizing for driver pointers or callbacks (common in 1990s C engines)
- Single-source-of-truth for hardware support declaration across entire codebase

**Historical note**: The commented-out `ASS_NoSound` suggests legacy support for "no audio" mode, likely removed or consolidated into a different handling path.

## Data Flow Through This File

1. **Runtime**: User selects sound card from menu → value stored in config
2. **Startup**: Audio init code reads config, matches `soundcardnames` value → switches to appropriate driver initialization (BLASTER_Init, ADLIBFX_Init, AWE32_Init, AL_Init, etc.)
3. **Runtime audio**: Selected driver's functions (BLASTER_Play, ADLIBFX_Play, AL_NoteOn, etc.) are called for audio output
4. **Version string**: ASS_VERSION_STRING ("1.04") likely reported in menus or debug output to identify audio subsystem version

## Learning Notes

- **1990s audio hardware abstraction**: Different sound cards required completely different initialization and I/O (DMA for SoundBlaster, port I/O for Adlib FM chip, MIDI for external devices)
- **Multi-driver pattern**: Each enum value maps to a different driver implementation file (a design still seen in modern audio engines, but now abstracted via plugin interfaces or dynamic loading)
- **Enum as dispatch key**: Using enums for multi-way dispatch is idiomatic for this era and avoids runtime string matching overhead
- The presence of so many distinct hardware targets (SoundBlaster, Adlib, UltraSound, SoundCanvas, etc.) reflects the fragmented DOS/early-Windows era gaming hardware landscape

## Potential Issues

- **Incomplete enumeration**: If new hardware is added to the engine, this header must be updated *and* a corresponding driver implementation added — no compile-time enforcement of this coupling
- **Deprecated hardware**: Many values (TandySoundSource, ProAudioSpectrum) reflect 1990s hardware unlikely to exist at runtime; cleanup risk if enum is pruned without checking all driver selection code
