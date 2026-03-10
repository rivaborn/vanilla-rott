# audiolib/source/_guswave.h — Enhanced Analysis

## Architectural Role

This header defines the **GUS (Gravis Ultrasound) wave PCM playback subsystem**, one of several parallel audio device drivers in the engine alongside BLASTER (Sound Blaster), ADLIBFX (AdLib FM), and MIDI/AWE32 synthesizers. GUS is specifically responsible for streaming sampled audio (Raw, VOC, WAV formats) with hardware voice scheduling and mixing. The volatile structures and hardware handle fields (`GF1voice`, `mem`) reveal direct coupling to the GUS card's on-board DSP, where the engine offloads voice mixing and playback—a key architectural advantage of GUS over simpler Sound Blaster cards.

## Key Cross-References

### Incoming (who depends on this file)
- **Audio manager layer** (likely in audiolib public API) coordinates device selection and calls GUSWAVE_Play, GUSWAVE_AllocVoice, GUSWAVE_GetVoice to manage voice lifecycle
- **Sound effect dispatcher** pulls playback status via GetSound callback and GUSWAVE_GetNextVOCBlock during audio frame service (typical interrupt or timer-driven cycle)
- **Game engine** may query GUSWAVE_GetVoice for handle-to-node lookup and volume/pan updates (Pan, Volume fields in VoiceNode)

### Outgoing (what this file depends on)
- **gus.c** (implied): Low-level GUS hardware initialization, register writes, DMA setup for on-card memory (`mem` field)
- **DMA subsystem** (audiolib/source/dma.h): Buffer transfer management for loading audio patches into GUS DRAM
- **Memory allocator**: VoiceNode structures allocated from heap or fixed pool; sound buffers pinned for DMA
- **libc/malloc**: Dynamic allocation of voice linked lists

## Design Patterns & Rationale

**Strategy pattern (callbacks)**: GetSound and DemandFeed function pointers abstract format handling—VOC blocks vs. raw PCM vs. demand-fed streams. This allows one voice node to handle multiple input formats without coupling to format parsers.

**Priority queue**: GUSWAVE_AllocVoice respects priority to steal voices when pool exhausted (VOICES=2 limit vs. MAX_VOICES=32). This is a **resource scarcity design** typical of 90s hardware—the GUS DSP can mix more than 2 voices, but the engine arbitrates them with priority (e.g., footsteps > ambience).

**Volatile hardware access**: The `volatile` qualifier on VoiceNode and voicestatus indicates memory-mapped hardware registers or DMA-visible buffers that the hardware modifies asynchronously. This prevents compiler optimizations that would break hardware synchronization.

## Data Flow Through This File

1. **Allocation**: Game calls GUSWAVE_AllocVoice(priority) → returns VoiceNode with hardware handle
2. **Setup**: Engine fills VoiceNode with sound pointer, sampling rate, format (wavedata type), loop parameters
3. **Playback**: GUSWAVE_Play() configures hardware (Pan, Volume, PitchScale) and starts DMA or hardware streaming
4. **Service loop**: Every ~10ms, engine calls voice->GetSound() callback → returns KeepPlaying or SoundDone
5. **Streaming**: For VOC, repeatedly calls GUSWAVE_GetNextVOCBlock() which updates NextBlock and BlockLength, advancing through the linked VOC chunks
6. **Cleanup**: When SoundDone returned, voice deallocated and returned to pool

## Learning Notes

This header exemplifies **hardware-centric audio design** specific to the DOS/early-Windows era. Key lessons:

- **GUS was premium**: Its on-board mixing DSP made it far superior to Sound Blaster (CPU-mixed PCM), justifying the separate subsystem despite code complexity.
- **Callback-driven streaming**: No high-level audio object model; instead, frames repeatedly pull data via callbacks—forcing the caller to manage buffer refills.
- **VOC format as legacy glue**: VOC block parsing (GUSWAVE_GetNextVOCBlock) suggests compatibility with older Blaster sound files; WAV/RIFF headers show newer format adoption.
- **Fixed voice limits**: The hardcoded VOICES=2 reflects GUS memory constraints, not DSP limits—a tradeoff between engine features and hardware resources.

Modern engines use **sample streaming libraries** (FMOD, Wwise) that hide hardware details; this code is the antithesis—every hardware register detail leaks into the data structure.

## Potential Issues

- **Voice starvation**: With only 2 concurrent voices (line 51), gameplay requiring >2 simultaneous sounds (gunfire + footsteps + ambient + music) will drop voices silently based on priority. No error reporting to caller if GUSWAVE_AllocVoice fails.
- **Commented-out code** (lines 48–51) suggests the engine author tested higher voice counts but reverted due to performance/stability. The discrepancy between VOICES and MAX_VOICES is unusual and hints at unfinished tuning.
- **No format validation**: The wavedata enum and GetSound callback assume the caller verified format compatibility; no runtime checks prevent mismatched format handlers from being called.
