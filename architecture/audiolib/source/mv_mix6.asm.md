# audiolib/source/mv_mix6.asm

## File Purpose
High-performance x86 assembly audio mixing kernels for the audio library. Implements six specialized functions to mix audio samples at different bit depths (8-bit, 16-bit) and channel configurations (mono, stereo, 1-channel interleaved). Uses self-modifying code and tight loop optimization for real-time audio performance.

## Core Responsibilities
- Mix source audio samples into a destination buffer via fractional sample position tracking
- Apply per-channel volume scaling using lookup tables (8-bit) or direct scaling (16-bit)
- Resample audio by advancing a fractional position counter by a per-sample rate increment
- Clip/clamp mixed samples to valid output ranges (via lookup table for 8-bit, conditional branches for 16-bit)
- Update global mixing state (destination write pointer, current playback position)
- Process samples in pairs (8-bit variants) or one at a time (stereo/16-bit) for instruction-level parallelism

## Key Types / Data Structures
None.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_MV_HarshClipTable` | DWORD (ptr to signed byte table) | global | Lookup table for clipping/limiting audio samples |
| `_MV_MixDestination` | DWORD (write cursor) | global | Current position in output buffer; updated after mixing |
| `_MV_MixPosition` | DWORD (fixed-point) | global | Current fractional playback position (integer.fraction format); advanced by rate each sample |
| `_MV_LeftVolume` | DWORD (ptr to signed word table) | global | Left channel volume scaling lookup table |
| `_MV_RightVolume` | DWORD (ptr to signed word table) | global | Right channel volume scaling lookup table |

## Key Functions / Methods

### MV_Mix8BitMonoFast_
- **Signature**: `void MV_Mix8BitMonoFast_()`
- **Purpose**: Mix 8-bit mono samples into mono destination, processing 2 samples per loop for efficiency
- **Inputs** (via registers):
  - `eax` = fractional playback position (16.16 fixed-point)
  - `edx` = sample rate increment
  - `ebx` = source buffer pointer
  - `ecx` = number of samples to mix (will be halved)
- **Outputs/Return**: None (side effects only)
- **Side effects**:
  - Updates `_MV_MixDestination` with final write position
  - Updates `_MV_MixPosition` with final fractional position
  - Self-modifying code: patches instructions at `apatch1–6` with volume table address, clipping table address, and rate increment
- **Calls**: None
- **Notes**: 
  - Processes samples in pairs to amortize branch cost
  - Uses lookup table for volume scaling and sample clipping (offset from `_MV_HarshClipTable + 128`)
  - Fractional position (`ebp`) is incremented by `edx` per sample; integer part extracted via `shr 16`

### MV_Mix8BitStereoFast_
- **Signature**: `void MV_Mix8BitStereoFast_()`
- **Purpose**: Mix 8-bit mono source into stereo destination with separate left/right volume
- **Inputs**: `eax` = position, `edx` = rate, `ebx` = source, `ecx` = sample count
- **Outputs/Return**: None
- **Side effects**: Updates `_MV_MixDestination`, `_MV_MixPosition`; patches `bpatch1–5`
- **Notes**:
  - One sample per loop (not paired)
  - Applies left volume via `bpatch1` and right volume via `bpatch2`
  - Output interleaved: [L][R][L][R]...

### MV_Mix8Bit1ChannelFast_
- **Signature**: `void MV_Mix8Bit1ChannelFast_()`
- **Purpose**: Mix 8-bit mono into 1-channel stereo output (16-bit word stride)
- **Inputs/Outputs**: Same as 8BitMono
- **Notes**: 
  - Processes 2 samples per loop
  - Destination stride is 2 bytes per sample (`add edi, 2`) instead of 1
  - Uses `epatch1–6` for self-modifying patches

### MV_Mix16BitMonoFast_
- **Signature**: `void MV_Mix16BitMonoFast_()`
- **Purpose**: Mix 16-bit signed mono samples with inline hard clipping
- **Inputs/Outputs**: Same register convention
- **Side effects**: Updates globals; patches `cpatch1–4`
- **Notes**:
  - Reads/writes 16-bit words
  - Clipping via conditional branches comparing against ±32768/32767 bounds (no lookup table)
  - Processes 2 samples per loop for efficiency

### MV_Mix16BitStereoFast_
- **Signature**: `void MV_Mix16BitStereoFast_()`
- **Purpose**: Mix 16-bit mono source into 16-bit stereo with separate volume per channel
- **Notes**:
  - One sample per loop
  - Inline clipping on each channel
  - Output interleaved; 4-byte stride per iteration

### MV_Mix16Bit1ChannelFast_
- **Signature**: `void MV_Mix16Bit1ChannelFast_()`
- **Purpose**: Mix 16-bit mono into 1-channel stereo output (32-bit word stride)
- **Notes**:
  - Processes 2 samples per loop
  - 8-byte destination stride (`add edi, 8`)
  - Patches `fpatch1–4`

## Control Flow Notes
All functions follow identical patterns:
1. **Setup phase**: Save registers, load global state pointers, apply self-modifying code patches to embed volume table addresses, clipping table addresses, and rate increment into the instruction stream
2. **Early exit**: Return if sample count is 0
3. **Prefetch**: Load first samples before entering main loop
4. **Main loop**: 
   - Fetch next source sample(s) via fractional index (`ebp >> 16`)
   - Apply volume scaling (table lookup for 8-bit; direct scaling for 16-bit)
   - Fetch current destination sample and add (mix)
   - Clip/limit result
   - Write back to destination
   - Increment fractional position by rate; decrement loop counter; branch if non-zero
5. **Finalization**: Store final mixing state back to globals, restore registers, return

These functions are pure mixing kernels called in a render loop from higher-level audio code (not shown).

## External Dependencies
- **External symbols**:
  - `_MV_HarshClipTable`: Defined elsewhere; used for 8-bit sample clipping via lookup
  - `_MV_MixDestination`: Global output buffer cursor
  - `_MV_MixPosition`: Global fractional sample position
  - `_MV_LeftVolume`: Defined elsewhere; 16-entry lookup table for left-channel volume
  - `_MV_RightVolume`: Defined elsewhere; 16-entry lookup table for right-channel volume

- **Assembler syntax**: TASM IDEAL mode; directives `p386`, `MODEL flat`, `MASM` compatibility; `ALIGN 4` loop labels for instruction cache alignment
