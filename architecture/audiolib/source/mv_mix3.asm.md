# audiolib/source/mv_mix3.asm

## File Purpose
Hand-optimized x86-32 assembly audio mixing routines for the sound engine. Provides fast mixing of 8-bit and 16-bit audio samples in mono and stereo configurations with sample-rate conversion via fractional position tracking and real-time clipping.

## Core Responsibilities
- Mix source audio samples into a destination buffer with volume scaling
- Perform sample-rate conversion using fractional position advancement
- Apply per-sample clipping to prevent overflow
- Support multiple audio formats: 8-bit/16-bit, mono/stereo, and 1-channel layouts
- Use self-modifying code patterns to inject runtime parameters (volume tables, clip tables, sample rates) into instruction immediates

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MixBufferSize | Constant (256) | File-static | Buffer size for mixing operations |
| {a,b,c,d,e,f}patch{1-6} | x86 Instruction immediates | File-static (self-modified) | Runtime-patched offsets for volume/clip tables and sample rates |

## Key Functions / Methods

### MV_Mix8BitMonoFast_
- **Signature:** `void MV_Mix8BitMonoFast_(eax=position, edx=rate, ebx=clipTable, edi=destBuf, esi=srcBuf, ecx=volTable)`
- **Purpose:** Mix 8-bit mono audio into a destination buffer with clipping via lookup table. Processes 2 samples per loop iteration.
- **Inputs:** Position (fixed-point), sample rate delta, harsh-clip table ptr, volume table ptr, source/destination buffers
- **Outputs/Return:** None (modifies destination buffer in-place)
- **Side effects:** Modifies edi/destination buffer; self-modifies code patches (apatch1-6) at init
- **Calls:** (none visible; this is a leaf function)
- **Notes:** Fixed-point arithmetic: position upper 16 bits index source; lower 16 bits are fractional. Harsh clip table is offset +128 to handle signed-to-unsigned conversion. Loop processes 256/2=128 pairs of samples.

### MV_Mix8BitStereoFast_
- **Signature:** `void MV_Mix8BitStereoFast_(eax=position, edx=rate, ebx=clipTable, edi=destBuf, esi=srcBuf, ecx=leftVolTable, [esp+8]=rightVolTable)`
- **Purpose:** Mix 8-bit stereo audio. Applies separate volume tables for left and right channels.
- **Inputs:** Position, rate, clip table, left volume table, right volume table (stack param), source/destination buffers
- **Outputs/Return:** None (in-place mix)
- **Side effects:** Modifies destination buffer; patches bpatch1-5 at init
- **Calls:** (none)
- **Notes:** Processes one stereo sample pair per iteration (2 bytes output). Right volume table passed on stack.

### MV_Mix8Bit1ChannelFast_
- **Signature:** `void MV_Mix8Bit1ChannelFast_(eax=position, edx=rate, ebx=clipTable, edi=destBuf, esi=srcBuf, ecx=volTable)`
- **Purpose:** Mix 8-bit single-channel audio. Similar to mono but processes with different destination stride (advances by 2 bytes instead of 1).
- **Inputs:** Position, rate, clip table, volume table, source/destination buffers
- **Outputs/Return:** None (in-place)
- **Side effects:** Modifies destination buffer; patches epatch1-6 at init
- **Calls:** (none)
- **Notes:** Loop advances destination by 2 bytes (`add edi, 2`), suggesting interleaved or padded single-channel layout.

### MV_Mix16BitMonoFast_
- **Signature:** `void MV_Mix16BitMonoFast_(eax=position, edx=rate, edi=destBuf, esi=srcBuf, ecx=volTable)`
- **Purpose:** Mix 16-bit mono audio with inline clipping (conditional branches for min/max, no lookup table).
- **Inputs:** Position, rate, volume table, source/destination buffers
- **Outputs/Return:** None (in-place)
- **Side effects:** Modifies destination buffer; patches cpatch1-4 at init
- **Calls:** (none)
- **Notes:** Uses comparison (`cmp eax, -32768 / 32767`) and conditional jumps for clipping instead of table lookup. Volume table is word-indexed (2*eax offset). Destination advances by 4 bytes per iteration.

### MV_Mix16BitStereoFast_
- **Signature:** `void MV_Mix16BitStereoFast_(eax=position, edx=rate, edi=destBuf, esi=srcBuf, ecx=leftVolTable, ebx=rightVolTable)`
- **Purpose:** Mix 16-bit stereo audio with separate volume tables per channel and inline clipping.
- **Inputs:** Position, rate, left/right volume tables, source/destination buffers
- **Outputs/Return:** None (in-place)
- **Side effects:** Modifies destination buffer; patches dpatch1-3 at init
- **Calls:** (none)
- **Notes:** One stereo pair per loop. Destination advances by 4 bytes per iteration.

### MV_Mix16Bit1ChannelFast_
- **Signature:** `void MV_Mix16Bit1ChannelFast_(eax=position, edx=rate, edi=destBuf, esi=srcBuf, ecx=volTable)`
- **Purpose:** Mix 16-bit single-channel audio with inline clipping. Destination stride is 8 bytes (2 samples).
- **Inputs:** Position, rate, volume table, source/destination buffers
- **Outputs/Return:** None (in-place)
- **Side effects:** Modifies destination buffer; patches fpatch1-4 at init
- **Calls:** (none)
- **Notes:** Similar to 16-bit mono but advances by 8 bytes per iteration, consistent with the 8-bit 1-channel variant.

## Control Flow Notes
All functions follow the same pattern:
1. **Init phase:** Self-modify code patches to inject runtime parameters (volume/clip tables, sample rates)
2. **Setup phase:** Initialize position and rate tracking; load first 1–2 samples
3. **Loop phase:** Fetch source sample(s), apply volume scaling, mix with destination, clip, write back, advance position
4. **Exit:** Return to caller

This is real-time audio mixing for a game engine, likely called repeatedly in a frame-update or audio callback context. No explicit init/shutdown is performed by these functions; their caller must provide pre-allocated buffers and tables.

## External Dependencies
- **Notable includes / imports:** None (pure assembly)
- **Defined elsewhere:** 
  - Volume lookup tables (passed as pointers)
  - Harsh clip tables (passed as pointers, offset by +128)
  - Source and destination audio buffers (passed as pointers)
  - All parameters are caller-provided; no global variables referenced
