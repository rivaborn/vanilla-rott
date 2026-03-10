# audiolib/source/mv_mix4.asm — Enhanced Analysis

## Architectural Role

This file implements the **core real-time audio mixing routines** that blend multiple PCM audio sources into a single output buffer, with per-sample volume scaling and dynamic resampling. It sits at the boundary between high-level mixer management (driver-level C code) and low-level hardware I/O. These functions are dispatched by the mixer scheduler based on source format (8/16-bit, mono/stereo/1-channel) and execute in real-time mixing callbacks, likely triggered by sound card DMA completion interrupts or application tick loops. The file is part of the larger `audiolib` subsystem (alongside BLASTER card control, MIDI synthesis, and FM synthesis modules).

## Key Cross-References

### Incoming (who depends on this file)
- **Mixer scheduler** (likely in a C mixing wrapper, e.g., `mv_*.c` or main mixer driver): Calls one of the six `MV_Mix*BitFast_` functions based on source audio format and channel configuration at mixing time
- **Global state managers** (elsewhere in audiolib): Initialize and update `_MV_LeftVolume`, `_MV_RightVolume` (volume lookup tables), `_MV_HarshClipTable` (clipping lookup), and pointers `_MV_MixDestination`, `_MV_MixPosition`

### Outgoing (what this file depends on)
- **Global audio state** (read/write):
  - `_MV_MixDestination`: Output buffer write pointer (updated per call to track position for next mix)
  - `_MV_MixPosition`: Fractional sample playback position (high 16 bits = index, low 16 = fraction; updated per sample for resampling)
  - `_MV_LeftVolume`, `_MV_RightVolume`: Lookup tables for per-sample volume mapping
  - `_MV_HarshClipTable`: Lookup table for fast 8-bit clipping (offset +128 to handle signed indices [-128, +127])
- **No function calls** (purely assembly, inline all operations for speed)

## Design Patterns & Rationale

**Self-Modifying Code (Peephole Runtime Patching):**
Each function patches 2–6 inline `mov` instructions (labeled `apatch1–6`, `bpatch1–5`, etc.) with the addresses of volume tables and rate increments before entering the hot loop. This eliminates register/memory indirection in the innermost loop, critical for 1990s CPU performance (pre-Pentium Pro out-of-order execution). Modern compilers achieve similar via loop unrolling and compiler hints; this is hand-tuned for bare-metal x86.

**Fractional Position Arithmetic:**
Position is stored as 32-bit fixed-point (upper 16 = integer index, lower 16 = sub-sample fraction). Rate (edx) is added each sample to advance position—elegant, allows arbitrary resampling ratios. This pattern is still used in modern engines (just implemented in C/SIMD now).

**Loop Unrolling (2 samples/iteration for 8-bit, dual-channel processing):**
Reduces loop overhead; paired sample fetches and mixing balance register usage and cache prefetch efficiency.

**Variant Explosion (6 functions for 3 format combinations):**
Rather than a single function with format dispatch inside the loop, separate entry points avoid runtime branching in the critical path. Trade-off: code size vs. speed.

**Table-Based vs. Inline Clipping:**
8-bit uses `_MV_HarshClipTable` lookup (faster for small tables, limited range), while 16-bit uses inline comparisons (cmp + conditional jumps). Reflects different trade-offs: 8-bit audio has only 256 value ranges; 16-bit would need a 64K table, so inline clamping is faster.

## Data Flow Through This File

```
Caller (mixer scheduler)
  ↓
  MV_Mix8Bit/16Bit[Mono|Stereo|1Channel]Fast_()
  ↓ (patched in: volume tables, clip tables, rate)
  ┌─ Init: load position (ebp), source ptr (esi), dest ptr (edi)
  │
  ├─ Loop:
  │   1. Fetch 2–4 source samples using fractional position (index = ebp >> 16)
  │   2. Apply volume via table lookup (mov [eax + volume_table_ptr])
  │   3. Mix: add destination sample
  │   4. Clip: lookup or inline cmp
  │   5. Write 1–2 bytes to destination
  │   6. Advance position: ebp += rate
  │   7. Decrement count, loop
  │
  ├─ Output: updated _MV_MixPosition (ebp) and _MV_MixDestination (edi)
  └→ Caller updates globals, proceeds to next source or I/O
```

**Key state transitions:**
- Position wraps naturally (no explicit check) if rate causes overflow—caller must validate source buffer size
- Destination pointer advances per sample (8-bit mono/stereo) or per 2–4 bytes (16-bit, 1-channel) depending on format
- Loop count implicit (ecx = 0 triggers exit); no guard for buffer overrun

## Learning Notes

**Idiomatic to this engine/era:**
1. **Self-modifying code** for optimization: Uncommon in modern x86-64; restricted in some architectures (ARM, RISC-V) and disabled in many modern OSes (W^X). Shows the desperation of 1990s real-time audio on limited CPUs.
2. **Fractional fixed-point position:** Still idiomatic; modern engines use the same principle (e.g., Godot, FMOD use fixed-point or floating-point position tracking).
3. **Assembly-level variant specialization:** Modern C/C++ + compiler vectorization auto-generates similar variants; hand-roll only for hotspots.
4. **No error handling:** Assumes valid buffers, no NULL checks, no overflow protection. Typical for embedded/DOS-era audio drivers.

**Connections to modern game engine concepts:**
- **Audio DSP node graph:** This is a single DSP node (resampler + mixer) in what would be a larger node graph in modern engines.
- **Real-time scheduling:** Called from a callback, must complete before next DMA tick (~10–50ms) to avoid glitching.
- **Volume/pan tables:** Precomputed lookups avoid sqrt, log during real-time mixing (still used in FMOD, Wwise).

## Potential Issues

1. **Cache coherency with code patching:** Self-modifying code can stall on modern CPUs due to instruction cache invalidation. Not a factor on 386/486, but problematic on Pentium+.
2. **No thread safety:** If mixer is called from multiple threads or a signal handler, concurrent patch operations would race. No synchronization primitives used.
3. **No bounds checking:** If position advances past source buffer or destination fills unexpectedly, silent buffer overrun / undefined behavior.
4. **Fixed sample rate per call:** Rate is patched once per function call; changing it mid-mix would require re-entry. Limits smooth pitch transitions without multi-call logic.
5. **Implicit position wraparound:** Upper 16 bits of ebp are assumed to stay within source buffer; overflow not detected. Caller must validate.

---

These routines represent a **high-performance bottleneck engineered for 1990s constraints**—every cycle counted. Modern engines trade some of that micro-optimization for portability, thread safety, and maintainability via higher-level languages and SIMD intrinsics.
