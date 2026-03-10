# audiolib/source/mvreverb.asm — Enhanced Analysis

## Architectural Role
This file implements low-level audio DSP primitives for the voice mixer's reverb/echo processing pipeline. The four exported functions are called during per-voice sample transformation in the active mixer update cycle—they sit between raw voice samples and the final mixed output, applying parametric reverb attenuation. These functions are part of the **MultiVoice** mixing subsystem's sample processing chain, positioned after pitch/rate adjustment and before final voice-to-output mixing.

## Key Cross-References

### Incoming (who depends on this file)
Based on the audio architecture, these PUBLIC functions are likely called by:
- **Mixer voice rendering loop** (in audiolib's mixing/voice management code) — during per-sample processing when reverb effect is active
- **Voice effect dispatcher** — routes active reverb voices to the appropriate bit-depth variant
- Called with frequency proportional to sample rate × active voice count (performance-critical path)

### Outgoing (what this file depends on)
- **No external symbols or subsystem calls** — pure x86 primitives with no dependencies
- Reads caller-supplied buffers (source audio) and volume/shift parameters
- Writes directly to destination buffer

## Design Patterns & Rationale

**Self-Modifying Code (SMC)** — `rpatch16` and `rpatch8` labels:
- Patches shift instructions at runtime with the reverb strength parameter from `ecx`
- Rationale: 1990s optimization to avoid register load inside tight inner loop; saves one register and one instruction per iteration
- Cost: SMC is slow on modern CPUs (pipeline flush, cache coherency overhead); incompatible with modern exploit mitigations (DEP/NX)
- Shows era-specific x86 optimization thinking

**Tight Loops with ALIGN 4**:
- Cache-line alignment for loop entry; reflects Pentium-era understanding of instruction fetch
- No branching inside inner loop except loop counter (minimal branch prediction misses)
- Registers pre-setup before loop entry

**Two Processing Paths per Bit-Depth**:
- Table-driven (`MV_*BitReverb_`): flexible, uses lookup tables for arbitrary transformations
- Shift-based (`MV_*BitReverbFast_`): faster for simple attenuation (geometric reduction), avoids memory lookup latency
- Trade-off: speed vs. flexibility

**Sign-Bias Handling** (`0x80`):
- PCM audio in this era often mixed unsigned (0–255 for 8-bit, 0–65535 for 16-bit)
- The `0x80` XOR/ADD operations convert between signed interpretation (for arithmetic) and unsigned representation (for storage)
- Particularly complex in `MV_8BitReverbFast_` where sign bit must be extracted, shifted, and re-added for proper rounding

## Data Flow Through This File

**MV_16BitReverb_** and **MV_8BitReverb_**:
```
Source buffer (raw 16/8-bit PCM)
    ↓
Volume table lookup per byte/sample
    ↓
0x80 bias adjustment (sign conversion)
    ↓
Destination buffer (attenuated PCM)
```

**MV_16BitReverbFast_** and **MV_8BitReverbFast_**:
```
Reverb shift parameter (ecx) → Runtime patch at rpatch label
Source buffer (raw PCM)
    ↓
Arithmetic right shift (specialized for this shift amount)
    ↓
0x80 bias / sign-aware rounding (8-bit variant only)
    ↓
Destination buffer (attenuated PCM)
```

The functions run synchronously during mixer polling; voice count × sample count iterations per audio frame.

## Learning Notes

**What modern engines do differently**:
- Avoid SMC (use SIMD with variable shift lane, or JIT compiler)
- Floating-point DSP internally (convert PCM → float, process, convert back)
- SIMD vectorization (process 4–16 samples per iteration, modern x86-64 SSE/AVX)
- Dynamic dispatch via function pointers or polymorphism, not SMC

**Idiomatic to this era**:
- Direct register-level optimization (eax/edx/esi/edi convention)
- Microsecond-scale audio buffers (real-time constraints on Pentium-class hardware)
- Assumption that assembly-language dominance for audio mixing was necessary

**Connection to engine concepts**:
- This is **effect stage in a simple effect-chain architecture** (no ECS, no modern graph-based effect routing)
- Stateless transforms (reverb = immediate attenuation, not convolution/impulse response)
- Per-sample processing loop (vs. modern frame-vectorized approaches)

## Potential Issues

1. **Self-modifying code incompatible with modern OS/CPU** — DEP/NX/SMM security features will block runtime code patching; modern mitigations assume code is immutable
2. **Cache coherency** — SMC flushes instruction cache; causes stalls on modern multi-core systems
3. **Instruction alignment fragility** — assumes specific x86 opcode lengths; relocatable code or different assembler output could break patch offsets
4. **Sign-bit arithmetic in 8-bit fast path** — complex rounding logic is error-prone; would benefit from unit tests (unclear if they exist)
5. **No bounds checking** — assumes caller provides valid buffer ranges and sample counts; buffer overrun will corrupt memory silently
