# audiolib/source/mv_mix.asm — Enhanced Analysis

## Architectural Role

This file implements the **real-time audio mixing engine** — the hot-path, sample-level processor that sits between the voice/channel orchestration layer and hardware output buffers (managed by BLASTER subsystem). Called once per audio frame, it resamples multiple voice streams into a single mono or stereo output, with per-channel volume control and automatic clipping. These functions are the innermost loop of the audio subsystem, directly feeding DMA buffers used by `BLASTER_BeginBufferedPlayback` and playback interrupts.

## Key Cross-References

### Incoming (who depends on this file)
- **Higher-level mixer orchestrator** (not visible in provided context, but inferred): Sets the global parameters (`_MV_SampleSize`, `_MV_LeftVolume`, `_MV_RightVolume`, `_MV_HarshClipTable`, `_MV_RightChannelOffset`, `_MV_MixPosition`, `_MV_MixDestination`) before each call, then invokes the appropriate `MV_Mix*Bit*_()` variant
- **Audio frame callback** (likely within BLASTER interrupt or timer handler): Calls these functions to fill the current DMA buffer half
- **Playback mode selectors** (e.g., BLASTER_SetMixMode): Determines which variant is used based on hardware format (8/16-bit, mono/stereo)

### Outgoing (what this file depends on)
- **Global mixing state** (read/write):
  - `_MV_MixDestination`: Current write pointer in output buffer
  - `_MV_MixPosition`: Fractional playback position (22.10 fixed-point)
- **Configuration globals** (read-only, pre-configured):
  - `_MV_LeftVolume`, `_MV_RightVolume`: Volume lookup tables (16-bit signed audio samples indexed by 8-bit sample value)
  - `_MV_HarshClipTable`: 256-entry clipping table for 8-bit (offset by 128 for signed range)
  - `_MV_SampleSize`: Sample width in source buffer (1 or 2 bytes)
  - `_MV_RightChannelOffset`: Byte distance from left sample to right in stereo buffers
- **No function calls**: Completely self-contained inner loops; no external dependencies

## Design Patterns & Rationale

**Self-Modifying Code (Code Patching):**
Instead of storing parameters in memory and loading them in the loop, the setup phase directly patches instruction operands (e.g., `mov [OFFSET apatch1+4], ebx`). This avoids:
- Extra memory loads in the tight loop
- Register pressure (all 7 general-purpose registers are actively used)
- Jump table overhead

Classic x86 optimization trade-off: upfront CPU cost (patching) for zero per-sample cost.

**Fractional Position with Bit-Shifting:**
Position stored as N.16 fixed-point (16-bit fractional part). Sample index extracted via `shr eax, 16`, which is faster than division. This enables pitch-accurate resampling without floating-point.

**Lookup Table Clipping (8-bit only):**
8-bit uses `_MV_HarshClipTable` (255-511 range, offset by 128): single memory load per sample. 16-bit uses explicit comparisons (`cmp/jge/jle` sequences) because values exceed 8-bit range.

**Unrolling & Latency Hiding:**
- 8-bit mono processes **2 samples/iteration** (fetches 4, mixes, writes in parallel)
- 8-bit stereo processes **1 stereo pair/iteration** (left+right interleaved)
- 16-bit similarly unrolled to hide multiplication/memory latencies

**Register Allocation:**
Manually optimized; every register is live and necessary:
- `esi` = source pointer
- `edi` = destination pointer
- `ebp` = fractional position (extended per-sample)
- `ecx` = sample count (loop counter)
- `eax`, `ebx`, `edx` = scratch for sample lookup, volume, mixing

## Data Flow Through This File

```
Setup Phase:
  [Caller sets globals]
  → Read _MV_SampleSize, volume tables, clip table, offsets
  → Code-patch XREF apatch{1-9}, bpatch{1-8}, cpatch{1-7}, dpatch{1-6}

Main Loop (per iteration):
  Position (ebp) + Rate (edx)
    ↓ (>> 16 to get index)
  Source Sample Fetch (byte/word from [esi + index])
    ↓ (2× eax for volume table offset)
  Volume Lookup [_MV_LeftVolume / _MV_RightVolume + (2×sample)]
    ↓ (signed result)
  Mix (add with destination sample at [edi])
    ↓
  Clip (table lookup for 8-bit; conditional branches for 16-bit)
    ↓
  Write Result [edi] → advance edi by sample width

Exit Phase:
  Store updated edi → _MV_MixDestination
  Store updated ebp → _MV_MixPosition
```

## Learning Notes

**What this teaches:**
- **Hand-optimized x86 assembly**: Every instruction choice matters; register allocation dominates performance
- **Fixed-point math**: Avoid floating-point in tight loops; use bit-shifting for division
- **Lookup tables as computation**: Pre-compute expensive operations (multiply-by-volume) into tables
- **Code generation & self-modification**: Rare in modern code but powerful for JIT-like scenarios

**Era-specific patterns (90s audio coding):**
- Direct hardware buffer access (no abstraction)
- Assembly-only critical path (C not fast enough at the time)
- Self-modifying code (relies on single-threaded execution; breaks modern instruction caches)
- Mono/stereo as separate code paths (not parameterized)

**Modern game engines do differently:**
- SIMD intrinsics (SSE, AVX) for parallel mixing
- Managed C++ or JIT-compiled hot paths instead of manual asm
- Parameterized mix loop (bit depth, channels as runtime variables, not code variants)
- Threaded or job-scheduled mixing (not frame-blocking)

## Potential Issues

- **Self-modifying code**: Incurs pipeline flushes on modern CPUs; instruction cache misses; breaks some security mitigations (W^X)
- **Single-threaded assumptions**: No interlocking on `_MV_MixPosition` or `_MV_MixDestination`; concurrent access would race
- **Hard-coded variants**: Adding a new bit depth (e.g., 24-bit) requires new assembly function; not maintainable at scale
- **Linear interpolation only**: Resampling is simple index lookup; no bandlimit filtering (aliasing at high pitches)
