# audiolib/source/mv_mix16.asm — Enhanced Analysis

## Architectural Role
This file is the **core sample mixing engine** for the MultiVoice audio subsystem. It implements four specialized inner-loop routines that real-time audio streams call to perform lossy mixing (combining multiple sources into a destination buffer) with per-channel volume scaling and overflow protection. These functions are the performance-critical path for all audio playback in the game engine—every sound effect and music sample flows through one of these routines during the main audio service interrupt.

## Key Cross-References

### Incoming (who depends on this file)
- **`BLASTER_SetMixMode()` and related BLASTER functions** (audiolib/source/blaster.c): Likely select and install the appropriate mixing routine variant based on detected hardware capabilities and mix format. These functions are the public API entry points that configure which `MV_Mix*` routine will be used during playback.
- **Audio service interrupt handler** (likely in blaster.c or a BLASTER-related source): Calls the selected mixing routine in a tight loop during DMA completion interrupts to fill the next audio buffer block. The routine address is probably stored in a function pointer patched at initialization.
- **Volume control subsystem**: Reads/updates `_MV_LeftVolume` and `_MV_RightVolume` tables. Functions like `BLASTER_SetVoiceVolume`, `AL_SetVoiceVolume`, etc. build these lookup tables and store their addresses in the globals this file uses.

### Outgoing (what this file depends on)
- **Global audio state** (defined externally, likely in blaster.c or multivc.c):
  - `_MV_HarshClipTable` — Precomputed lookup table for 8-bit overflow clipping (indexed -128…127 → clipped value)
  - `_MV_MixDestination` — Output buffer write pointer (updated in-place by this function)
  - `_MV_MixPosition` — 16.16 fixed-point playback position (updated per sample)
  - `_MV_LeftVolume` / `_MV_RightVolume` — Pointer to per-channel volume scaling tables
  - `_MV_SampleSize` — Sample stride (1 or 2 bytes)
  - `_MV_RightChannelOffset` — Stereo channel separation in bytes
- **No external function calls** — Pure assembly, no dependencies on other functions. All work is self-contained.

## Design Patterns & Rationale

**Self-Modifying Code ("Patching")**: The dominant pattern. Runtime configuration values (volume table pointers, sample rates, clip tables) are written directly into instruction immediates at labels `apatch1`–`apatch9`, `bpatch1`–`bpatch8`, etc. This eliminates conditional branches from the inner loop, reducing latency and pipeline stalls—critical for real-time audio at 44.1 kHz or higher on 1990s hardware.

**Fixed-Point Arithmetic (16.16 format)**: Position tracking uses 16-bit integer + 16-bit fraction (`ebp` register). Samples are fetched by right-shifting the position by 16 bits (`shr ebp, 16`) to get the sample index. This supports sample-rate conversion (playing faster/slower than original) without floating-point overhead.

**Lookup Table Volume Scaling**: 8-bit samples use a 256-entry volume table (one per possible sample value); the volume-translated value is fetched in a single instruction (`movsx eax, byte ptr [2*eax+12345678h]`). This trades memory (small table) for speed (no multiplication).

**Dual Sample Processing (8-bit routines)**: `MV_Mix8BitMono16_` processes two samples per loop iteration (`shr ecx, 1` doubles the effective count). This overlaps instruction latencies and reduces loop overhead, but makes the code harder to follow.

**Harsh Clipping Strategy**: 
- **8-bit**: Lookup table (`_MV_HarshClipTable + 128`) allows any byte value to be instantly clipped
- **16-bit**: Direct comparison (`cmp eax, -32768` / `cmp eax, 32767`) with conditional jumps—simpler than a table, still fast

## Data Flow Through This File

```
INPUT:
  eax (ebp) ──> 16.16 fixed-point playback position
  edx        ──> Sample rate delta per sample
  ebx (esi)  ──> Source audio buffer (start address)
  ecx        ──> Number of samples to mix
  
SETUP PHASE:
  Read _MV_LeftVolume, _MV_RightVolume → patch into loop
  Read _MV_HarshClipTable, _MV_SampleSize, _MV_RightChannelOffset → patch
  edi ← _MV_MixDestination (destination write pointer)
  
MAIN LOOP (per sample or sample pair):
  1. Calculate index: shr position >> 16 → sample index
  2. Fetch: [esi + index] → get source sample
  3. Volume: sample → lookup table → scaled value
  4. Read destination: [edi] → existing sample
  5. Mix: scaled + existing → sum
  6. Clip: harsh clip on overflow
  7. Write: [edi] ← clipped result
  8. Advance position: ebp += rate delta
  9. Advance destination: edi += stride (1, 2, or 4 bytes)
  
OUTPUT:
  _MV_MixDestination ← final edi (for next call)
  _MV_MixPosition ← final ebp (playback position)
  Destination buffer modified in-place
```

## Learning Notes

**1. Idiomatic 1990s Real-Time Audio Programming:**
   - Fixed-point math avoids FPU latency and precision loss
   - Self-modifying code was accepted practice for inner loops; modern compilers do similar via inline specialization
   - Lookup tables for non-linear ops (volume, clipping) instead of computation
   - Careful register allocation and instruction ordering to maximize parallelism on superscalar CPUs (Pentium/Pentium Pro era)

**2. Sample Rate Conversion (Resampling):**
   - The 16.16 position format allows playback at any rate without resampling filters. Each sample is only used once (no interpolation), so quality degrades at extreme pitch shifts—acceptable for game audio where pitch shifts are moderate.

**3. Mono vs. Stereo Trade-offs:**
   - Mono routines process two samples per loop (halve loop count)
   - Stereo routines process one stereo pair per loop but apply *two* volume tables (left + right)
   - This reflects the mixing engine's assumption that stereo is more common in games

**4. Contrast with Modern Engines:**
   - Modern game audio (Wwise, FMOD) use SSE/NEON SIMD and avoid self-modifying code
   - They often apply interpolation or higher-quality resamplers
   - Configuration is typically dynamic (not patched into code)

## Potential Issues

1. **No bounds checking on source position**: The code assumes `position >> 16` always falls within the source buffer. A malformed position (e.g., corruption via unrelated bug) could cause buffer overread. This is acceptable if the caller guarantees valid position, but fragile if the contract breaks.

2. **Self-modifying code fragility**: If TASM or the assembler changes instruction encoding, the patch offsets (e.g., `apatch1+4`) will silently break. A length-checking assertion would help, but this is era-typical.

3. **Synchronization gap**: The position and destination globals are not atomically updated. If an interrupt fires between updating `_MV_MixDestination` and `_MV_MixPosition`, a concurrent reader sees inconsistent state. For single-threaded systems (DOS/early Windows), this is acceptable; for later OS ports, it may be a race condition.

4. **Hard-coded offsets in 16-bit routines**: Lines like `inc ebx` (after loading `_MV_LeftVolume`) and `inc esi` assume the volume table is consecutive in memory. This works but is implicit—a comment would improve maintainability.

---

**Sources for cross-references inferred from:** Cross-Reference Index excerpt (BLASTER subsystem, volume control functions).
