# audiolib/source/multivoc.h — Enhanced Analysis

## Architectural Role

MULTIVOC is the **primary voice-mixing abstraction layer** in the audio subsystem, sitting between the game engine and hardware-specific audio drivers (BLASTER, GUS, ADLIBFX, etc.). It provides a uniform multi-voice polyphonic mixer API that abstracts away hardware differences, manages scarce voice resources through priority-based allocation, and handles format-agnostic playback of WAV, VOC, and raw audio streams. The game engine (rott/) calls MULTIVOC exclusively for all sound effects and music; MULTIVOC internally delegates to hardware drivers.

## Key Cross-References

### Incoming (who depends on this file)
- **rott/ game engine**: All sound playback calls (SFX, music, ambient sounds) route through MULTIVOC API
- Implied by presence of `callbackval` (unsigned long) — game passes sound instance IDs for completion notifications
- Priority-based voice allocation (`MV_VoiceAvailable(priority)`) suggests game engine manages sound priority hierarchy

### Outgoing (what this file depends on)
- **Hardware drivers** (BLASTER, GUS, ADLIBFX, AWE32): MV_Init selects soundcard; implementation delegates mixing/output to hardware-specific .c files
- **DMA/IRQ subsystem** (audiolib/source/dma.c, irq.c): Error codes MV_DMAFailure, MV_DMA16Failure, MV_IrqFailure suggest hardware initialization failures
- **Format parsers** (implicit in MULTIVOC.C): WAV/VOC header parsing, sample rate/bit depth detection
- **Memory management** (MV_LockMemory, MV_UnlockMemory): DOS-era memory locking for DMA safety

## Design Patterns & Rationale

**Voice-Handle Pool Pattern**: Each active sound is a "voice" with an integer handle (≥ MV_MinVoiceHandle = 1). Game stores handles, uses them to control active sounds (pitch, pan, kill). Handles isolate game from internal voice state management.

**Hardware Abstraction Pattern**: MV_Init accepts abstract `soundcard` ID; implementation probes and selects appropriate driver (BLASTER, GUS, etc.). Errors (MV_UnsupportedCard, MV_BlasterError, etc.) expose hardware failures.

**Priority-Based Voice Stealing**: `MV_VoiceAvailable(priority)` returns handle only if voice exists at this priority level. This allows graceful degradation: low-priority sounds are killed when voice budget is exhausted.

**Callback Registration**: `callbackval` (unsigned long) parameter is opaque data passed to game's completion callback (set via MV_SetCallBack). Allows game to track which sound finished without additional state.

**Format Abstraction**: MV_PlayWAV, MV_PlayVOC, MV_PlayRaw hide format parsing; game passes raw file pointer. MV_InvalidWAVFile and MV_InvalidVOCFile errors bubble up parsing failures.

**3D Audio via Stereo Projection**: MV_Pan3D(angle, distance) converts 3D coords to stereo pan. Simpler than true 3D (no HRTF), but sufficient for 90s action game.

**Reversible Stereo**: MV_SetReverseStereo allows in-game stereo inversion (accessibility feature).

## Data Flow Through This File

```
Game Engine (rott/)
  → MV_Init(soundcard, rate, voices, channels, bits)
        ↓ probes hardware, allocates voice buffers
        ↓ delegates to BLASTER/GUS/etc drivers
  → MV_PlayWAV/VOC/Raw(data, params, priority, callbackval)
        ↓ finds available voice, parses format
        ↓ starts DMA transfer via hardware driver
        ↓ returns voice handle
  → [Game stores handle]
  → MV_SetPan(handle, ...) / MV_SetPitch(handle, ...)
        ↓ updates active voice parameters
  → MV_Kill(handle)
        ↓ stops voice, deallocates
  → [Game's callback fired when voice completes]
  → MV_Shutdown() → stops all voices, releases hardware
```

**Reverb State**: MV_SetReverb / MV_SetReverbDelay affect ALL voices globally (no per-voice reverb), suggesting simple shared reverb buffer or post-processor.

**Recording Parallel**: MV_StartRecording accepts callback; mixer captures input alongside playback output—rare for 90s games, likely used for in-game voice chat or audio diagnostics.

## Learning Notes

**Voice-Based vs. Stream-Based Mixing**: Unlike modern game engines (Wwise, FMOD), MULTIVOC is voice-centric—each sound claims a fixed-size voice slot. This fits DOS/early-90s hardware where per-voice DMA and synthesis were real constraints. Modern engines use sample-accurate event streams.

**Hardware Diversity Abstraction**: The 1994 PC audio landscape was fragmented (SoundBlaster, Gravis UltraSound, Roland, Yamaha). MULTIVOC demonstrates why HALs exist: a single codebase adapts to many boards. This era's lesson: assume heterogeneous hardware; modern lesson: abstract even high-level APIs (e.g., OpenAL wraps platform audio).

**Callback-Based Completion**: No polling for "is voice done"; game registers a callback. This is efficient for interrupt-driven hardware and predates callbacks becoming mainstream in language design. Modern engines use promises/async, but the pattern is the same.

**Priority Over Latency**: No emphasis on low-latency streaming (MV_StartDemandFeedPlayback is exception). Most sounds are pre-loaded (MV_PlayWAV assumes memory buffer). Reflects that 1995 games queued sounds frames in advance, not in real-time response to player.

**3D Audio Simplicity**: No HRTF, impulse responses, or distance-based filtering; just stereo panning. Even 3D games of this era used crude spatial audio. Modern engines apply distance attenuation, frequency roll-off, etc.

## Potential Issues

- **No voice starvation guarantee**: If all voices are busy and game requests sound at low priority, MV_VoiceAvailable returns error, forcing game to suppress the sound. No graceful degradation strategy is codified here.
- **Callback race conditions** (if MV_SetCallBack is called while voices are active): Unclear if callback is atomic or if mid-playback changes race. DOS single-threaded environment mitigates this, but worth noting.
- **Format parsing in MV_PlayWAV/VOC**: If file pointer is invalid or truncated, behavior is undefined (implementation-dependent). No validation hints in header.
- **Memory locking overhead**: MV_LockMemory/MV_UnlockMemory are expensive DOS DPMI operations; calling these frequently could stall. No guidance on when to call them.
