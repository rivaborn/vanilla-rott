# audiolib/source/mv_mix5.asm — Enhanced Analysis

## Architectural Role
This file implements the core PCM sample mixing primitives that accumulate individual voice/sample playback into the master output buffer. It operates as a leaf subsystem in the audio pipeline: higher-level code (voice managers, playback schedulers) invokes these functions to mix active samples at runtime. The six variants (8-bit/16-bit × mono/stereo/1-channel) act as optimized kernels for the most common mixing scenarios, minimizing per-sample overhead in the critical audio path where CPU time directly impacts frame rate and audio quality.

## Key Cross-References

### Incoming (who depends on this file)
- **Voice/sample playback handlers**: Called from code that services active voice objects and sample playback requests (inferred from globals `_MV_MixPosition` and rate parameter usage)
- **Audio service routine** (likely in BLASTER or mixer manager): Dispatches to the appropriate `MV_Mix*BitFast_` variant based on source format and destination channel configuration
- **Global state consumers**: Code reading `_MV_MixPosition` after mixing completes to update playback position; code reading `_MV_MixDestination` to know where the next sample writes begin

### Outgoing (what this file depends on)
- **Volume tables** (`_MV_LeftVolume`, `_MV_RightVolume`): Pre-computed signed 16-bit lookup tables (indexed by 8-bit sample value) that apply per-channel gain without per-sample multiply
- **Clipping table** (`_MV_HarshClipTable`): 256-entry lookup for 8-bit harsh clipping (offset +128 to handle signed range)—reflects design choice to use lookup for 8-bit, branch clamping for 16-bit (likely cost model from target CPU)
- **Mixer state variables**: `_MV_MixDestination`, `_MV_MixPosition` read/written as global interchange points with the calling mixer loop

## Design Patterns & Rationale

### Self-Modifying Code (apatch/bpatch/cpatch/dpatch/epatch/fpatch)
Runtime instruction patching injects frame-specific values (volume table address, rate scale, clip table base) into immediate operands. **Why?** Avoids register pressure: all 6 general-purpose registers are committed to loop-critical operations (position, rate, source/dest pointers, samples, clipping tables). Dynamically modifying code is cheaper than loading immediates or recomputing addresses in the loop. The TASM comment hints this was a known workaround for assembler limitations.

### 16.16 Fixed-Point Arithmetic
Sample position stored as 16-bit integer + 16-bit fractional part (ebp >> 16 extracts index). **Rationale:** Avoids floating-point hardware overhead (which was slow/rare in 1990s) while enabling sub-sample pitch shifting and smooth rate interpolation. Adding `edx` (rate) per sample implements linear interpolation.

### Separate Optimized Paths
Rather than a single flexible mixer, six hardcoded variants optimize for common (format, channel) pairs. **Tradeoff:** Binary size and maintenance cost vs. zero branching overhead and tight unrolled loops per case.

### Harsh Clip vs. Branch Clipping
8-bit mixing uses lookup table; 16-bit uses branch-based min/max clamping. **Rationale:** 8-bit table fits L1 cache (~256 bytes); branch predictor amortizes branch cost for 16-bit comparisons across iterations.

## Data Flow Through This File

```
Input globals:
  - _MV_MixPosition (16.16 fixed-point playback position)
  - _MV_MixDestination (write cursor in output buffer)
  - _MV_LeftVolume / _MV_RightVolume (gain tables)
  - _MV_HarshClipTable (clipping LUT for 8-bit)

Per invocation:
  eax: position in source
  edx: rate (pitch scale per sample)
  ebx: source buffer base
  ecx: sample count to mix

Core loop (two samples per iteration):
  1. Extract integer part: edx >> 16 → index into source
  2. Fetch byte/word from source[index]
  3. Translate via volume table → signed value
  4. Add to current destination sample
  5. Apply clipping (table or branch)
  6. Write result to destination
  7. Advance position: ebp += edx (rate)
  
Output globals:
  - _MV_MixPosition ← final ebp (updated playback position)
  - _MV_MixDestination ← final edi (next write position)
```

## Learning Notes

- **Era-specific optimization**: Self-modifying code, fixed-point math, unrolled loops, and register-constrained kernels were industry standard before dynamic compilation and SIMD instructions. This file is a textbook example of 1990s console/DOS game audio engineering.
- **Contrast with modern audio**: GPU mixing, SIMD mixing kernels (using AVX/NEON), and floating-point sample rates have made this approach obsolete. Most modern engines use middleware (FMOD, Wwise) that abstracts these details.
- **Interpolation strategy**: The simple add-per-sample approach (no table lookup for rates) works because rate is constant within a mix batch; more sophisticated engines use hermite or sinc interpolation for higher quality.
- **Mono vs. stereo separation**: The code copies logic across variants rather than unifying it—reflects constraints of 1990s assembler development and the performance-critical nature of the task.

## Potential Issues

- **No bounds checking**: Functions assume the caller has allocated sufficient destination buffer space. Buffer overflow is silent and corrupts audio/memory.
- **Hardcoded strides**: Mono advances by 1 byte (8-bit) or 2 bytes (16-bit), 1-channel by 2/4, stereo by 2/4—inflexible for formats like 5.1 surround or unusual layouts.
- **Self-modifying code cache coherency**: On modern CPUs with split I-cache, runtime patching (mov [eax], ebx) may not flush instruction cache; could cause stale code execution if scheduler doesn't serialize properly.
- **Volume table assumption**: Code assumes `_MV_LeftVolume` and `_MV_RightVolume` point to 256-entry signed word tables. Mismatch would silently read garbage.
