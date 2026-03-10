# audiolib/source/mv_mix4.asm

## File Purpose
Low-level x86 assembly audio mixing routines that blend source audio samples into a destination buffer with volume scaling and sample-rate conversion. Supports 8-bit and 16-bit PCM with mono, stereo, and single-channel layouts using runtime code patching for performance optimization.

## Core Responsibilities
- Mix source audio samples into destination buffer with per-sample volume translation
- Handle sample-rate conversion via fractional position tracking (fixed-point arithmetic)
- Apply harsh clipping to prevent audio distortion (table-based for 8-bit, inline comparisons for 16-bit)
- Support six audio format variants: 8/16-bit × mono/stereo/1-channel
- Optimize hot loops through self-modifying code injection of runtime parameters

## Key Types / Data Structures
None (pure assembly, no C-style type definitions).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _MV_HarshClipTable | DWORD | external | Lookup table for clipping out-of-range audio values (offset +128 for signed indexing) |
| _MV_MixDestination | DWORD | external | Current write pointer into output audio buffer |
| _MV_MixPosition | DWORD | external | Fractional sample position for resampling (upper 16 bits are integer index) |
| _MV_LeftVolume | DWORD | external | Volume scaling lookup table for mono/left channel |
| _MV_RightVolume | DWORD | external | Volume scaling lookup table for right channel |

## Key Functions / Methods

### MV_Mix8BitMonoFast_
- Signature: `void (eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 8-bit mono source into stereo destination with volume and resampling.
- Inputs: eax (fractional position), edx (rate increment per sample), ebx (source buffer base), ecx (sample count)
- Outputs/Return: Updates global _MV_MixDestination (edi) and _MV_MixPosition (ebp)
- Side effects: Self-modifies code at apatch1–6 labels; reads/writes global audio buffers; advances destination pointer
- Calls: (via patched instructions) _MV_LeftVolume table lookup, _MV_HarshClipTable lookup
- Notes: Processes 2 samples per loop iteration (ecx halved). Fractional position stored in ebp (upper 16 bits = index, lower = fraction). Uses pushad/popad for register preservation.

### MV_Mix8BitStereoFast_
- Signature: `void (eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 8-bit stereo source with independent left/right volume scaling.
- Inputs: eax (fractional position), edx (rate increment), ebx (source buffer), ecx (sample count)
- Outputs/Return: Updates _MV_MixDestination and _MV_MixPosition
- Side effects: Patches bpatch1–5; modifies global buffers; advances destination by 1 byte per sample
- Calls: (patched) _MV_LeftVolume, _MV_RightVolume lookups
- Notes: Processes 1 sample per loop (no count shift). Applies left and right volume tables independently before clipping.

### MV_Mix8Bit1ChannelFast_
- Signature: `void (eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 8-bit single-channel audio with 2-byte destination stride.
- Inputs: eax (fractional position), edx (rate increment), ebx (source buffer), ecx (sample count)
- Outputs/Return: Updates _MV_MixDestination and _MV_MixPosition
- Side effects: Patches epatch1–6; modifies global buffers; advances destination by 2 bytes per sample
- Calls: (patched) _MV_LeftVolume, _MV_HarshClipTable
- Notes: Differs from Mono variant in destination stride (edi+=2 vs. edi+=1). Processes 2 samples per loop.

### MV_Mix16BitMonoFast_
- Signature: `void (eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 16-bit mono source with word-sized samples and inline clipping.
- Inputs: eax (fractional position), edx (rate increment), ebx (source buffer), ecx (sample count)
- Outputs/Return: Updates _MV_MixDestination and _MV_MixPosition
- Side effects: Patches cpatch1–4; modifies global buffers; advances destination by 4 bytes per iteration
- Calls: (patched) _MV_LeftVolume word-indexed lookups (offset: 2×eax)
- Notes: Replaces table-based clipping with inline comparisons (cmp eax, ±32768). Processes 2 samples per loop.

### MV_Mix16BitStereoFast_
- Signature: `void (eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 16-bit stereo with per-channel volume and inline clipping.
- Inputs: eax (fractional position), edx (rate increment), ebx (source buffer), ecx (sample count)
- Outputs/Return: Updates _MV_MixDestination and _MV_MixPosition
- Side effects: Patches dpatch1–3; modifies global buffers; advances destination by 4 bytes per sample
- Calls: (patched) _MV_LeftVolume, _MV_RightVolume (word-indexed)
- Notes: Processes 1 sample per loop. Applies inline clipping to both channels independently.

### MV_Mix16Bit1ChannelFast_
- Signature: `void (eax=position, edx=rate, ebx=start, ecx=count)`
- Purpose: Mix 16-bit single-channel audio with 4-byte destination stride and inline clipping.
- Inputs: eax (fractional position), edx (rate increment), ebx (source buffer), ecx (sample count)
- Outputs/Return: Updates _MV_MixDestination and _MV_MixPosition
- Side effects: Patches fpatch1–4; modifies global buffers; advances destination by 8 bytes per iteration
- Calls: (patched) _MV_LeftVolume (word-indexed)
- Notes: Processes 2 samples per loop with 8-byte stride (vs. 4 for stereo). Inline clipping.

## Control Flow Notes
These routines are called by the audio subsystem's real-time mixer to blend audio sources into an output buffer. Each function:
1. Saves all registers (pushad)
2. Initializes pointers and loads runtime parameters via code patching (volume tables, clip tables, sample rates injected at labels apatch/bpatch/cpatch/etc.)
3. Pre-fetches first input samples using fractional position
4. **Main loop**: fetch destination sample(s) → apply volume scaling → mix with source → apply clipping → write result → advance fractional position → loop
5. Updates global position and destination pointers for next call
6. Restores registers (popad)

Variant selection depends on source bit-depth and channel configuration; 8-bit variants use table-based clipping, 16-bit use inline comparisons.

## External Dependencies
- **Assembler directives**: `.386`, `.MODEL flat`, `USE32` (80386 protected-mode 32-bit segments)
- **External symbols** (defined elsewhere in audio library):
  - `_MV_HarshClipTable` – lookup table for 8-bit clipping
  - `_MV_MixDestination` – output buffer write pointer
  - `_MV_MixPosition` – fractional sample playback position
  - `_MV_LeftVolume` – mono/left-channel volume table
  - `_MV_RightVolume` – right-channel volume table
