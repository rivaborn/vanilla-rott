# audiolib/source/_multivc.h — Enhanced Analysis

## Architectural Role
This private header defines the core mixing layer for MULTIVOC, a hardware-agnostic digital audio subsystem that bridges the game engine and platform-specific sound drivers (BLASTER, ADLIB, AWE32, GUS). The VoiceNode structure and static mixing functions implement a fixed 8-voice polyphonic mixer with support for multiple audio formats and real-time effects. It abstracts away hardware differences so the game engine treats all sound cards identically.

## Key Cross-References

### Incoming (who depends on this file)
- **multivoc.c** (private implementation) — Only caller of all static functions; implements public API that the game engine uses
- **Game engine** (rt_*.c) — Indirectly through MULTIVOC's public API; doesn't see these internals
- Voice data providers: Game asset loaders (VOC/WAV readers) populate VoiceNode structures

### Outgoing (what this file depends on)
- **BLASTER subsystem** (blaster.h) — DMA setup and interrupt handling for Sound Blaster playback
- **Format-specific decoders** — VOC/WAV/Raw block readers (likely in multivoc.c or separate codec modules)
- **Hardware I/O** (inp/outp) — Port-level access to VGA (ATR_INDEX, STATUS_REGISTER_1) for diagnostics
- **System memory** — Ring buffer management and voice pool allocation (pre-allocated at init)

## Design Patterns & Rationale

### Strategy Pattern (Function Pointers)
Each VoiceNode carries two function pointers:
- `voice->GetSound()` — Format-specific next-block loader (VOC, WAV, Raw, DemandFeed)
- `voice->mix()` — Format-specific mixer (8BitMono, 16BitStereo, etc.)

**Why?** Avoids conditional branches per-sample. In 1994, CPU predictors didn't exist; branch mispredicts were expensive. By pre-selecting the correct mixer at voice creation, the inner loop has zero branching.

### Pre-calculated Lookup Tables
`VOLUME8` and `VOLUME16` (256-entry tables) store scaled samples for each volume level.

**Why?** Multiplication was slower than table lookup on 486/Pentium CPUs. Modern CPUs reverse this, but in 1995 this was essential. The comment `// MIX_VOLUME` macro shows an alternate approach was considered but rejected.

### Fixed Voice Pool
Exactly 8 voices, no dynamic allocation during playback.

**Why?** Real-time audio forbids allocation/deallocation during mixing (memory fragmentation, latency). Allocate once at init. If 9 sounds play, the lowest-priority voice is preempted—a reasonable tradeoff for a game.

### Doubly-Linked List (VList)
VoiceNode forms a linked list with next/prev pointers.

**Why?** Efficient insertion/removal at arbitrary positions (voice preemption), O(1) given a pointer. Cache-friendly for sequential iteration during mixing.

### Callback-based Block Fetching
Instead of buffering entire sounds, `GetSound()` feeds chunks on-demand.

**Why?** Enables streaming from disk without loading entire samples into RAM. Critical for low-memory DOS systems (640 KB base RAM typical).

### DemandFeed Voices
Voices can supply data via a callback (`DemandFeed` function pointer), not pre-loaded samples.

**Why?** Allows procedural/dynamic audio without pre-recording. Used for generated sounds or compressed codecs.

## Data Flow Through This File

### Voice Lifecycle
```
Allocation → Configuration → Active List → Mixing Loop → End/Preemption → Deallocation
```

1. **Allocation** (`MV_AllocVoice`)
   - Game requests sound with priority
   - If all 8 slots full, find lowest-priority voice and preempt
   - Return available VoiceNode

2. **Configuration** (implicit in calling code)
   - Set `voice->sound`, `voice->length`, `voice->SamplingRate`, `voice->bits`
   - Set `voice->wavetype` → determines `GetSound` callback
   - Call `MV_SetVoiceMixMode()` → selects format-specific `voice->mix` function
   - Set volume/pan via `voice->LeftVolume`/`voice->RightVolume` (pointers to lookup tables)
   - Call `MV_PlayVoice()` → insert into active list

3. **Mixing Loop** (during `MV_ServiceVoc()`)
   - For each active voice in VList:
     - Call `MV_Mix(voice, buffer_index)`
     - `MV_Mix` → calls `voice->mix()` (format-specific)
     - Format mixer reads from `voice->sound + voice->position`
     - Scales samples via volume tables
     - Writes to output mix buffer
     - Advances `voice->position`

4. **Block Fetching** (when data runs out)
   - Call `voice->GetSound()` to load next block
   - Handles loop-back via `voice->LoopStart`/`voice->LoopCount`
   - Returns `KeepPlaying` or `NoMoreData`

5. **End of Life**
   - `GetSound` returns `NoMoreData`
   - `MV_StopVoice()` removes from active list
   - Voice handle becomes available for reallocation

### Output Buffer Ring
- **Total size:** 4096 samples (256 × 16 buffers)
- **Chunk size:** 256 samples per mixing iteration
- **Purpose:** Decouple mixing rate from hardware interrupt rate
- **Hardware:** DMA reads from one buffer while MULTIVOC fills the next

## Learning Notes

### 1990s Game Audio vs. Modern Engines

**Idiomatic to this era:**
- **Fixed voice limits** — No dynamic allocation (hard real-time constraint)
- **Lookup table optimization** — Faster than multiply on 486/Pentium
- **Assembly inner loops** — C compilers generated slower code; hand-optimized x86 was necessary
- **Interrupt-driven mixing** — Sound card interrupts trigger servicing
- **Callback blocks** — Streaming without full buffering; typical for DOS/early Windows

**Modern engines do differently:**
- **Unbounded voice allocation** — Efficient allocators make this safe
- **Float arithmetic** — GPUs excel at it; CPUs have fast FPUs now
- **Compiler-optimized mixing** — Modern C++ compilers + SIMD intrinsics outperform hand-written assembly
- **Event-driven (not interrupt-based)** — Main loop queries sound state; OS handles threading
- **Streaming abstraction** — Full-featured audio frameworks (e.g., FMOD, Wwise) handle this transparently

### Conceptual Connections
- **No ECS here** — This is a simple object pool + linked list, not entity-component architecture
- **Similar to software rasterizers** — Multiple "active entities," dispatched via function pointers, iterated in tight loops
- **Pitch shifting via fixed-point scaling** — `PitchScale`/`RateScale` resampling without floating point; classic signal processing
- **Reverberation in the mix stage** — Effects aren't separate; `MV_*BitReverb` functions process samples alongside mixing

## Potential Issues

### 1. **Concurrent Access (Race Conditions)**
VList modifications (adding/removing voices) and mixing iterations happen in the same data structure. If `MV_ServiceVoc()` runs asynchronously (interrupt handler), and game code calls `MV_PlayVoice()` or `MV_StopVoice()` from main loop, there's no visible locking. Likely relies on DOS single-threading assumption or critical sections (not shown in header). **Modern concern:** Modern audio engines use lock-free data structures or explicit synchronization.

### 2. **Fixed Voice Preemption**
Only 8 voices. If game plays 9 simultaneous sounds, the 9th triggers preemption of the lowest-priority voice. If priority tuning is poor, critical sounds (e.g., player footsteps) could be silenced by ambient effects. No priority enforcement visible here; depends on caller discipline.

### 3. **Buffer Synchronization**
Ring buffer management assumes precise coordination between MULTIVOC (writer) and hardware DMA (reader). If the hardware reads past the write pointer, audio glitches occur. Not visible in this header, but implicit in `TotalBufferSize` and `NumberOfBuffers` constants.

### 4. **Pitch Scale Overflow**
`PitchScale` is `unsigned long`. Extreme pitch shifts could overflow without checks visible in this header. Likely handled in `MV_SetVoicePitch()` in multivoc.c.

### 5. **DOS Hardware Assumptions**
- `inp()`/`outp()` for port I/O (DOS/Win9x only)
- `#pragma aux` for x86 inline assembly (Watcom C specific; won't compile on modern GCC/Clang without adaptation)
- ATR_INDEX/STATUS_REGISTER_1 registers are VGA-specific
- **Non-portable:** This code is tightly bound to x86 DOS/Windows 3.1–95 platforms.

### 6. **No Sample Rate Mismatch Handling**
If `voice->SamplingRate` ≠ hardware playback rate, resampling happens implicitly via `RateScale`. No explicit error if the ratio is extreme (e.g., 1:100), which could cause buffer underruns.

---

This header reveals the 1994–95 state-of-the-art in real-time audio: fixed-pool voice management, lookup-table optimization, x86 assembly for critical paths, and callback-based streaming. Its design would be considered unnecessarily low-level by modern standards, but was essential for shipping a 60-FPS game on a 133 MHz Pentium with 8 MB of RAM.
