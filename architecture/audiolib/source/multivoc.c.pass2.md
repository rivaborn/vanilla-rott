# audiolib/source/multivoc.c — Enhanced Analysis

## Architectural Role

**multivoc.c** is the **central audio mixing engine** of the Apogee audio subsystem, serving as the exclusive implementation of multi-voice sampled sound playback across all supported DOS sound cards. It bridges game code (which calls `MV_PlayWAV`, `MV_PlayVOC`, etc.) to hardware abstractions (Sound Blaster, Gravis UltraSound, Pro Audio Spectrum) via a unified voice allocation and interrupt-driven circular buffer mixer. All other audio modules (blaster.c, guswave.c, etc.) are card-specific drivers that multivoc.c coordinates; multivoc.c is the only file providing the `MV_*` public API.

## Key Cross-References

### Incoming (who depends on this file)
- **Game code** (from rott/ subsystem) calls `MV_PlayWAV`, `MV_PlayVOC`, `MV_PlayRaw`, `MV_PlayVOC3D`, `MV_PlayWAV3D` to trigger sound playback
- **Main audio init** (likely rt_main.c, rt_game.c) calls `MV_Init` and `MV_Shutdown`
- **Audio menu/config** calls `MV_SetVolume`, `MV_SetReverb`, `MV_GetVolume` to manage playback settings
- **Voice management** from game loop calls `MV_Kill`, `MV_KillAllVoices`, `MV_VoicesPlaying` to stop sounds

### Outgoing (what this file depends on)
- **Card drivers** (blaster.c, guswave.c, pas16.c, sndscape.c, sndsrc.c): Each driver implements card-specific init, playback setup, and interrupt servicers; multivoc.c calls their init functions and hooks their interrupt handlers
- **Low-level DMA** (dma.c): `DMA_GetCurrentPos` to read current playback position for buffer boundary detection
- **Interrupts** (interrup.h): `DisableInterrupts`, `RestoreInterrupts` for interrupt-safe voice list modifications
- **DPMI** (dpmi.h): `DPMI_LockMemory`, `DPMI_GetDOSMemory` to lock code/data and allocate DOS-addressable buffers
- **Pitch** (pitch.h): `PITCH_GetScale` for fixed-point rate scaling during playback
- **Linked lists** (linklist.h): `LL_SortedInsertion`, `LL_Add`, `LL_Remove` for voice pool and active voice list management
- **User hooks** (usrhooks.h): Custom memory allocators

## Design Patterns & Rationale

| Pattern | Where | Why |
|---------|-------|-----|
| **Interrupt-driven circular buffer** | `MV_ServiceVoc()` called from hardware interrupt | DOS real-time requirement: CPU cannot block waiting for DMA; DMA fires interrupt when buffer boundary is reached; mixer prepares next buffer during this interrupt |
| **Voice pool + linked list** | `VoiceList`, `VoicePool` | Avoid allocation during real-time mixing; pre-allocate all voices at init; keep active voices in sorted order (by priority) for preemption |
| **Function pointers (GetSound, mix)** | `voice→GetSound`, `voice→mix`, `MV_MixFunction` | Polymorphism without virtual tables: different file formats (WAV, VOC, raw, demand-feed) have different block-fetch logic; different output formats (8/16-bit mono/stereo) have different mixing kernels |
| **Lookup tables** | `MV_VolumeTable[64][256]`, `MV_PanTable[32][64]` | Real-time performance: pre-calculate volume scaling and stereo panning at init to avoid multiplication in the tight mixing loop (which runs in interrupt context) |
| **Priority-based preemption** | `MV_AllocVoice()` with `LL_SortedInsertion` | Graceful degradation: when all voices allocated, steal the lowest-priority playing voice rather than silently dropping the request |
| **Memory locking** | `MV_LockMemory`, `DPMI_LockMemory` | Interrupt safety: lock mixing code (`MV_Mix`, etc.) and data (buffers, lookup tables) to physical RAM to prevent page faults during interrupt handler execution |
| **Global mixing state** | `MV_MixDestination`, `MV_LeftVolume`, `MV_RightVolume`, `MV_MixPosition` | Performance: avoid passing large structures through call stacks in real-time path; instead, globals cache the "current mix context" |

**Structural rationale**: The architecture reflects DOS-era constraints:
- No kernel-managed audio subsystem → custom interrupt handler (`MV_ServiceVoc`)
- Limited RAM → pre-allocated fixed-size voice pool instead of dynamic allocation
- Hardware-specific details → abstraction layer for each card (but multiplexed through multivoc.c's public API)
- DMA-based playback → circular buffers and interrupt-driven pacing

## Data Flow Through This File

```
Application
    ↓
MV_PlayWAV/VOC/Raw/DemandFeed (init voice)
    ↓
MV_AllocVoice (grab from pool or preempt)
    ↓
MV_SetVoicePitch, MV_SetVoiceVolume (configure)
    ↓
MV_PlayVoice (add to VoiceList, sorted by priority)
    ↓
[Hardware DMA fires interrupt at buffer boundary]
    ↓
MV_ServiceVoc (interrupt handler)
    ├─ Read DMA position → MV_MixPage (current buffer index)
    ├─ Apply reverb (if enabled) by mixing delayed buffer
    └─ For each voice in VoiceList:
        ├─ MV_Mix (core mixer loop)
        │  ├─ Call voice→GetSound to fetch next block
        │  ├─ Call voice→mix (8/16-bit mono/stereo kernel) to apply pitch/volume/pan
        │  └─ Advance voice→position (fixed-point)
        ├─ If voice finished (GetSound → NoMoreData):
        │   ├─ MV_StopVoice (move to VoicePool)
        │   └─ Call user callback
        └─ Loop next voice
    ↓
Next buffer ready; DMA plays it
    ↓
[Repeat]
    ↓
MV_Kill / MV_KillAllVoices (stop voice, fire callback)
    ↓
MV_Shutdown (free all, stop DMA)
```

**Key state transitions**:
- Voice: `Free (in VoicePool)` → `Allocated` → `Playing (in VoiceList)` → `Free`
- Buffer: `Silence` → `Mixed (by MV_Mix for each voice)` → `Played by DMA`

## Learning Notes

### Idiomatic to 1990s DOS Audio Engines
1. **Interrupt-driven circular buffers**: Modern OS-level audio APIs (WASAPI, ALSA, Core Audio) handle buffering; DOS required hand-coded interrupt handling
2. **Pre-allocated voice pools**: No dynamic allocation during playback (would cause unpredictable latency/fragmentation); voice count fixed at init time
3. **Fixed-point arithmetic**: `voice→RateScale` uses fixed-point for pitch-shifting without floating-point overhead (slow on 386/486)
4. **Lookup-table effects**: Reverb, volume, and pan computed via pre-calculated tables rather than real-time math; trades memory for speed
5. **Memory locking**: DPMI memory locking (locking code/data to physical RAM) to prevent page faults in interrupt handlers—modern OS provides virtual memory with interrupts disabled, no such concern

### Concepts in Modern Game Engines
- **ECS**: Modern engines decouple sound logic from game state; multivoc.c couples voice state tightly (via voice handles) to game control
- **Streaming**: No support for streaming audio from disk; all sounds must fit in DOS memory. Modern: load-on-demand, buffer pools
- **DSP/GPU mixing**: Modern engines offload mixing to DSP or GPU; multivoc.c mixes in CPU interrupt context (CPU-heavy)
- **Reverb/effects**: Modern engines use convolution or algorithmic reverb; multivoc.c uses simple delay-based reverb via circular buffer wrap-around
- **3D audio**: `MV_PlayWAV3D` uses pre-calculated pan tables indexed by (angle, distance); modern engines use HRTF filtering or real-time binaural processing

### Connections to Game Engine Programming
- **Voice allocation as resource management**: Similar to texture/memory pools in graphics engines
- **Priority-based preemption**: Same pattern as draw-call sorting or priority queues in CPU scheduling
- **Circular buffers for lock-free data**: Common in real-time systems (audio, networking, rendering)
- **Function pointers for codec pluggability**: Early form of polymorphism; modern: plugin architectures, trait objects (Rust), interfaces (C#/Java)

## Potential Issues

1. **Memory coupling**: All voices, buffers, and lookup tables allocated as one contiguous DOS memory block (`MV_BufferDescriptor`); fragmentation or size miscalculation causes init failure. Modern approach: separate allocations with better error recovery.

2. **No error recovery in interrupt handler**: `MV_ServiceVoc` runs at hardware interrupt level; if any voice→mix or voice→GetSound crashes (e.g., null pointer, invalid data), the system hangs. No try-catch equivalent in C.

3. **Voice handle collision risk**: `MV_VoiceHandle` increments without reset; after ~2 billion voices allocated, it could theoretically wrap and collide with old handles (unlikely in practice, but theoretically possible).

4. **Reverb buffer wraparound logic**: The reverb delay calculation (`source += MV_BufferLength` when `source < MV_MixBuffer[0]`) assumes buffers are contiguous; if buffer layout changes, it silently breaks.

5. **No per-voice error state**: If a GetSound callback returns an invalid pointer or length, `MV_Mix` has no way to detect and stop the voice gracefully; it will mix garbage or crash.

6. **Single global reverb level**: All voices share one reverb setting; cannot apply per-voice reverb or selectively reverb only certain sounds (e.g., indoor vs. outdoor acoustic environments in a game level).

---

**First-pass analysis** covered the file's extensive API and internal mechanics exhaustively. This second pass highlights how multivoc.c anchors the audio subsystem, abstracts hardware diversity, and exemplifies real-time interrupt-driven mixer design for a resource-constrained platform (DOS).
