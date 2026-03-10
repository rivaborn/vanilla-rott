# audiolib/source/mv_mix3.asm — Enhanced Analysis

## Architectural Role

This file is the **performance-critical core of the multi-voice mixer** in the Vanilla ROTT audio subsystem. It provides optimized x86-32 assembly implementations for mixing multiple audio streams into a final output buffer, supporting multiple sample formats (8/16-bit) and channel configurations (mono/stereo/1-channel). These routines are called repeatedly by higher-level mixer orchestration code (likely `multivc.c` or similar) during each audio callback/frame, making them part of the real-time DSP pipeline that feeds into hardware backends (Sound Blaster, AdLib, AWE32, GUS).

## Key Cross-References

### Incoming (who depends on this file)
- **Multi-voice mixer framework** (inferred from file naming: `multivc.c` or variant) — likely selects and invokes one of these six functions based on audio format and channel layout
- **Audio backend drivers** (BLASTER, AdLib, AWE32, GUS from cross-reference index) — indirectly depend on mixed output
- **Voice/channel management code** — allocates and configures voice state that feeds parameters into these functions

### Outgoing (what this file depends on)
- **Caller-provided lookup tables:** Volume tables (one per channel), harsh-clip tables (for non-linear clipping)
- **Caller-provided buffers:** Source (decoded audio samples) and destination (mixed output buffer)
- **Runtime parameter injection:** Sample rate multiplier (`edx` register), fixed-point position tracking (`ebp`), buffer pointers
- **No external library calls** — pure algorithmic mixing

## Design Patterns & Rationale

**Self-Modifying Code Pattern:** Each function patches immediate operands in its own instruction stream (e.g., `apatch1: movsx eax, byte ptr [eax+12345678h]`) at initialization. This avoids register/memory indirection in the tight loop and reflects 1990s performance optimization discipline for audio DSP.

**Dual/Quad Processing Strategy:** 8-bit mono/1-channel processes **2 or 4 samples per loop iteration** (`mix8Mloop`, `mix81Cloop`); 16-bit processes **2 samples per iteration**. This balances instruction pipeline efficiency (Pentium era) with branch misprediction cost.

**Format-Specific Clipping:**
- **8-bit:** Harsh-clip via **lookup table** (offset +128 to map signed sample indices)
- **16-bit:** Harsh-clip via **inline conditional branches** (cmp/jge/jle), trading table memory for pipeline bubbles

This reflects the cost/benefit tradeoff: 8-bit samples benefit from table lookup; 16-bit overhead makes branching acceptable.

**Fixed-Point Resampling:** Position tracked as 32-bit fixed-point (upper 16 bits = integer index, lower 16 bits = fraction). Advancing by `edx` per sample enables variable playback rates without per-sample division.

## Data Flow Through This File

```
Input:  Source samples (8/16-bit, decoded audio stream)
        + Position (16.16 fixed-point)
        + Rate delta (sample rate conversion coefficient)
        + Volume table (lookup for amplitude scaling)
        + Clip table (lookup for non-linear saturation)
        + Destination buffer (output mix)

Transform:
  1. Unpack position → sample index (upper 16 bits)
  2. Fetch source[index] and source[index+1/2/3]
  3. Apply volume scaling via table lookup
  4. Add to destination (mixing)
  5. Apply clip table / inline clipping
  6. Write back to destination
  7. Advance position by rate delta
  8. Loop for 128–256 samples

Output: Destination buffer (modified in-place with mixed audio)
```

## Learning Notes

**Idiomatic to this era/engine:**
- Self-modifying code was a valid (though controversial) optimization in the 1990s; modern engines use function pointers, SIMD intrinsics, or JIT compilation instead
- Fixed-point arithmetic (16.16) reflects pre-FPU ubiquity; modern DSP uses floating-point
- Harsh-clip via table (8-bit) shows memory hierarchy awareness: CPU cache misses are cheap vs. register pressure
- No DMA or interrupt-driven streaming visible here; this is synchronous, frame-based mixing (called from audio callback)

**Game engine concepts:**
- This implements **voice mixing** (one of many audio sources) → a **master output buffer**
- The "harsh clip table" is non-linear saturation: maps [-128, 127] → finite range, preventing overflow distortion
- **Sample-rate conversion via fractional position** is classic **linear interpolation** without the interpolation—only the integer-indexed lookup (alias-prone, but fast for 1990s hardware)

## Potential Issues

1. **Alias Distortion Risk:** Resampling uses nearest-neighbor (integer index only), not linear/cubic interpolation. Downsampling source at high rate without filtering may introduce aliasing artifacts.

2. **Self-Modifying Code Fragility:** Instruction cache coherency is CPU-dependent (Pentium era could have issues). Modern processors with instruction TLBs may require `cpuid` serialization or code cache flushes (not present here).

3. **Fixed-Point Overflow:** If volume tables contain large values or mixed samples accumulate before clipping, intermediate `eax + edx` could overflow 32-bit bounds. Clipping is per-pair, so cumulative mixing across multiple voices relies on caller discipline.

4. **No Stereo Pan in Mono:** `MV_Mix8BitMonoFast_` uses single volume table; cannot express per-voice pan (would require separate L/R tables). Panning must happen upstream in multi-voice orchestration.
