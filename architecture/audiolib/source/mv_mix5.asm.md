# audiolib/source/mv_mix5.asm

## File Purpose
Low-level assembly implementation of high-performance audio sample mixing functions. Provides six variants for mixing 8-bit and 16-bit audio in mono, stereo, and single-channel configurations with real-time volume control and sample-rate interpolation.

## Core Responsibilities
- Mix source audio samples into destination buffer with rate-based interpolation
- Apply volume scaling via pre-computed lookup tables (separate for left/right channels)
- Clip/clamp mixed samples to prevent distortion (harsh clip table for 8-bit, min/max clamping for 16-bit)
- Dynamically patch instruction immediates at runtime for zero-overhead parameter passing
- Process multiple samples per loop iteration (typically two at a time) for throughput
- Maintain and update global playback position and output buffer pointers

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _MV_HarshClipTable | DWORD (pointer) | global | 256-entry lookup table for 8-bit sample clipping |
| _MV_MixDestination | DWORD (pointer) | global | Current write position in output buffer |
| _MV_MixPosition | DWORD (pointer) | global | 16.16 fixed-point playback position in source |
| _MV_LeftVolume | DWORD (pointer) | global | Left channel volume translation table |
| _MV_RightVolume | DWORD (pointer) | global | Right channel volume translation table |

## Key Functions / Methods

### MV_Mix8BitMonoFast_
- Signature: `void MV_Mix8BitMonoFast_(eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 8-bit mono samples with interpolation and clipping
- Inputs: eax (fractional position), edx (pitch rate), ebx (source buffer), ecx (sample count)
- Outputs/Return: None (state updated in globals)
- Side effects: Modifies _MV_MixDestination and _MV_MixPosition; self-modifying code (apatch1–6 immediates)
- Calls: None
- Notes: Processes two samples per loop iteration; harsh clip via table lookup; uses 16.16 fixed-point interpolation

### MV_Mix8BitStereoFast_
- Signature: `void MV_Mix8BitStereoFast_(eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 8-bit stereo samples with independent left/right volumes
- Inputs: eax, edx, ebx, ecx (as above)
- Outputs/Return: None
- Side effects: Modifies globals; self-modifying code (bpatch1–5)
- Calls: None
- Notes: One stereo pair (L+R) per iteration; stride = 2 bytes in destination

### MV_Mix8Bit1ChannelFast_
- Signature: `void MV_Mix8Bit1ChannelFast_(eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 8-bit single-channel samples at 2-byte strides
- Inputs: eax, edx, ebx, ecx
- Outputs/Return: None
- Side effects: Modifies globals; self-modifying code (epatch1–6)
- Calls: None
- Notes: Destination advance = 2 bytes per sample (edi+2); similar structure to mono variant

### MV_Mix16BitMonoFast_
- Signature: `void MV_Mix16BitMonoFast_(eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 16-bit mono samples with branch-based clipping to [−32768, 32767]
- Inputs: eax, edx, ebx, ecx
- Outputs/Return: None
- Side effects: Modifies globals; self-modifying code (cpatch3–4); conditional jumps for clipping
- Calls: None
- Notes: Two samples per loop; clipping via compare/jump rather than lookup table; 4-byte destination stride

### MV_Mix16BitStereoFast_
- Signature: `void MV_Mix16BitStereoFast_(eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 16-bit stereo samples with per-channel clipping
- Inputs: eax, edx, ebx, ecx
- Outputs/Return: None
- Side effects: Modifies globals; self-modifying code (dpatch1–3)
- Calls: None
- Notes: One L+R pair per iteration; 4-byte destination stride

### MV_Mix16Bit1ChannelFast_
- Signature: `void MV_Mix16Bit1ChannelFast_(eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 16-bit single-channel samples at 4-byte strides
- Inputs: eax, edx, ebx, ecx
- Outputs/Return: None
- Side effects: Modifies globals; self-modifying code (fpatch1–4)
- Calls: None
- Notes: Two samples per loop; destination advance = 8 bytes (edi+8); four samples (4×2 bytes) processed

## Control Flow Notes
All six functions follow this pattern:
1. **Init**: Save registers (pushad), load global pointers, patch immediate values in code (apatch*/bpatch*/cpatch*/dpatch*/epatch*/fpatch*)
2. **Loop setup**: Initialize fractional position and fetch first samples via table lookup
3. **Main loop** (ALIGN 16): Fetch source at interpolated position (ebp >> 16), volume-translate (table lookup), mix with destination, apply clipping, advance position by rate (edx), write output
4. **Exit**: Store final position and destination pointer to globals, restore registers (popad), return

Invoked during audio frame processing to accumulate voice/music playback into the mix buffer.

## External Dependencies
- **Externals** (defined elsewhere): `_MV_HarshClipTable`, `_MV_MixDestination`, `_MV_MixPosition`, `_MV_LeftVolume`, `_MV_RightVolume`
- No #include or imports; pure 32-bit x86 assembly

**Technical notes**: Self-modifying code patterns use OFFSET patching to inject runtime values into immediates, avoiding register pressure. Comment "convice tasm to modify code" suggests TASM assembler compatibility workaround. Hardcoded placeholder immediates (12345678h) are replaced at runtime.
