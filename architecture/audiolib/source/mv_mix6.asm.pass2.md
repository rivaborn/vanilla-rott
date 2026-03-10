# audiolib/source/mv_mix6.asm — Enhanced Analysis

## Architectural Role

This file implements the **real-time mixing kernel layer** of the audio engine—the innermost performance-critical path for voice mixing. Each function is a specialized mixer for a specific sample format (8-bit/16-bit) and channel configuration (mono/stereo), called during the main audio render loop (likely from BLASTER or other audio backend drivers) to composite multiple voice samples into the output buffer. The six variants allow the audio system to dispatch to the fastest code path for each voice's active configuration.

## Key Cross-References

### Incoming (who depends on this file)
- **Audio backend drivers** (BLASTER_*, GUS, ADLIBFX subsystems based on cross-ref index) call these mixing kernels during interrupt-driven or polling-based audio render callbacks
- **Higher-level mixing code** (not visible in cross-ref excerpt) wraps these kernels with state management (voice allocation, effect chains, voice priority)
- Global state pointers (`_MV_LeftVolume`, `_MV_RightVolume`, `_MV_HarshClipTable`) are populated by initialization/configuration functions (likely in `mv_mix.c` or similar)

### Outgoing (what this file depends on)
- **Global read/write**: 
  - `_MV_MixDestination` (output buffer cursor, incremented each render cycle)
  - `_MV_MixPosition` (fractional playback position, advanced per-sample; shared across voices in same render callback)
  - `_MV_LeftVolume`, `_MV_RightVolume` (lookup table pointers, set once at init/configuration change)
  - `_MV_HarshClipTable` (clipping/limiting lookup for 8-bit, set once at init)
- **No calls to other functions**—purely computational

## Design Patterns & Rationale

**Self-Modifying Code**: Each function patches 2–6 `MOD` instructions with actual table addresses and rate increments at entry. This avoids register indirection overhead in the inner loop, moving address loads outside the loop. This pattern was common in 1990s real-time audio (e.g., SoundBlaster drivers) to squeeze every CPU cycle. Modern CPUs with branch prediction and cache complexity make this less effective.

**Fixed-Point Fractional Positioning**: The `ebp` register holds fractional position as a 16.16 fixed-point value. Advance per sample (`add ebp, edx`), extract integer part for indexing (`shr ebp, 16`). This avoids floating-point overhead and ensures bit-exact resampling determinism.

**Lookup-Table Clipping (8-bit) vs. Conditional Clipping (16-bit)**: 
- 8-bit variants use `_MV_HarshClipTable` (offset by +128 to handle signed range [−128,127])
- 16-bit variants use conditional jumps to clamp to [−32768, 32767]
- Rationale: 8-bit table fits in L1 cache; 16-bit branching avoids table memory and misprediction cost for typically in-range samples

**Two-Sample Loop Unrolling** (in mono/1-channel 8/16-bit variants): Processes pairs of samples per iteration, reducing branch overhead by ~2×. Prefetches first sample before loop entry to hide initial latency.

## Data Flow Through This File

1. **Setup Phase** (on entry): 
   - Snapshot globals into registers (`_MV_LeftVolume` → `ebx`; `_MV_HarshClipTable` + 128 → `ebx`)
   - Patch instruction immediates with these values (write to `OFFSET apatchN+offset`)
   
2. **Main Loop** (repeated per sample or sample pair):
   - **Fetch**: Use fractional position (`ebp >> 16`) to index into source buffer
   - **Volume Scale**: Table lookup or direct scaling on fetched sample
   - **Read Destination**: Fetch current output buffer sample
   - **Mix**: Add scaled source to destination
   - **Clip/Limit**: Lookup or conditional clamp to valid range
   - **Write**: Store back to destination; advance destination pointer
   - **Advance Position**: Increment `ebp` by rate; loop until sample count exhausted

3. **Finalization**: Store final `_MV_MixDestination` and `_MV_MixPosition` back to globals for next render cycle

## Learning Notes

**Idiomatic to 1990s Audio DSP**: This code exemplifies DOS/early-Windows real-time audio optimization—manual register allocation, self-modifying code, fixed-point arithmetic, lookup tables. Modern engines use SIMD (SSE, AVX, NEON) and higher-level abstractions.

**No State Return**: Functions return nothing; all results are side effects (globals updated). This is typical of low-level ISR-safe code that avoids stack overhead.

**Resampling via Fractional Indexing**: The fractional position technique is foundational to digital audio resampling (used in all modern engines for pitch shifting and variable-rate playback).

**Dispatch by Format**: The six variants (2 sample depths × 3 channel configs) show early 1990s dispatch by specialization rather than SIMD—each kernel is hand-optimized for its case. Modern engines would use loop SIMD or vectorized intrinsics.

## Potential Issues

- **Thread Safety**: Global state (`_MV_MixPosition`, `_MV_MixDestination`) is read/written without synchronization. This works on single-threaded DOS/early-Windows but breaks if audio render runs concurrently with game logic.
- **Cache/Branch Prediction**: Self-modifying code undermines CPU prefetch and BTB effectiveness; modern processors often disable self-modifying code optimizations for security/reliability.
- **Unverified Assumptions**: No overflow/underflow checks on fractional position; overflow wraps silently at 32-bit boundary (could cause audible glitches on long playback).
- **Hardcoded Clipping**: 16-bit clipping uses fixed bounds; no dynamic range control (e.g., normalized mixing or per-channel peak tracking).
