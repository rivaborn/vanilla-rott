# audiolib/source/mv_mix1.asm — Enhanced Analysis

## Architectural Role
This file implements the core inner loop for the real-time audio mixing subsystem, specifically optimized for 8-bit mono playback via Sound Blaster hardware. It bridges the high-level voice/channel mixing abstraction and the low-level DMA buffer operations: multiple audio sources (synthesized FM voices, sampled sounds, MIDI) converge through lookup-table-based volume/clipping transformations into a single PCM buffer during ISR servicing. The self-modifying code pattern was critical for DOS-era performance constraints where every CPU cycle mattered during interrupt-driven audio.

## Key Cross-References
### Incoming (who depends on this file)
- **BLASTER audio driver** (`audiolib/source/blaster.c`): Likely called from the DMA interrupt handler during `BLASTER_ServiceInterrupt` to mix active voices into the playback buffer
- **MultiVoice mixer infrastructure**: The `MV_` naming convention suggests integration with a larger mixer that manages multiple concurrent audio channels
- **Higher-level playback functions** (from BLASTER API): `BLASTER_BeginBufferedPlayback`, `BLASTER_SetPlaybackRate` set parameters that feed into this routine

### Outgoing (what this file depends on)
- **Lookup tables** (caller-provided via registers): Volume table (ecx) and harsh clip table (ebx) — these are initialized by the caller, likely in C code during audio setup
- **DMA ring buffer memory** (edi): Output destination managed by the BLASTER driver
- **Source sample data** (esi): Typically from a loaded sound effect or synthesized PCM stream

## Design Patterns & Rationale
**Self-modifying code** (apatch1–apatch6): Runtime patching of immediate operands avoids the register/memory load overhead of passing parameters through the calling convention. In a tight 256-sample loop running during ISR, this saves ~6–8 CPU cycles per iteration on a 486/Pentium.

**Fixed-point fractional positioning** (`ebp` stores position, upper 16 bits = integer index): Enables pitch shifting and arbitrary resampling by varying the rate increment (`edx`). Caller controls playback speed by changing this parameter.

**Lookup-table transformations**: Volume scaling and harsh clipping via direct memory indexing rather than arithmetic operations — aligned with audio DSP best practices for real-time systems.

**Exactly 256-sample chunks**: Matches typical DMA buffer granularity and simplifies caller's buffer management logic.

## Data Flow Through This File
```
Caller (ISR handler) registers:
  eax=position_frac, edx=rate_increment, ecx→volume_table, ebx→clip_table
                ↓
            Self-patch code with runtime parameters
                ↓
        Loop 256 times:
          1. Read source sample at (position >> 16)
          2. Apply volume via lookup: volume_table[sample]
          3. Blend with destination via clip table: clip_table[volume_sample + dest]
          4. Advance fractional position by rate
                ↓
        Modified destination buffer (in-place DMA write)
        (Position not returned — caller must maintain across calls)
```

## Learning Notes
**Legacy optimization idiom**: Self-modifying code was standard in 1990s real-time audio/graphics code before aggressive CPU instruction caches made this pattern less useful. Modern engines use inline functions or SIMD intrinsics instead.

**Fixed-point arithmetic**: The Q16.16 (or similar) format is foundational to game audio resampling. This predates floating-point DSP libraries and modern sample-rate conversion algorithms.

**Harsh clipping design**: The +128 offset in the clip table index allows signed table lookups — a compact way to represent both negative overflow and positive overflow in a single byte lookup. This prevents audio crackling from saturating at buffer boundaries during multi-voice mixes.

**Single-threaded ISR assumption**: No synchronization primitives; relies on being called only from the audio ISR, never from main thread simultaneously.

## Potential Issues
- **No bounds checking**: Fractional position overflow could cause out-of-bounds source reads if rate is large or position wraps. Caller responsible for overflow handling.
- **Harsh clip table assumptions**: The table must be exactly 512 bytes (256 for each signed sample value + 128 offset). Misalignment or wrong size would corrupt mixing.
- **Self-modifying code hazards**: CPU instruction cache invalidation not explicit; could cause stale code execution on modern CPUs or in concurrent scenarios (though ISR model prevents this in practice).
- **No stereo support**: Hard-coded for 8-bit mono; separate routine needed for stereo or 16-bit sources.
