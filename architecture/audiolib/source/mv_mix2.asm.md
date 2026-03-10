# audiolib/source/mv_mix2.asm

## File Purpose
Hand-optimized x86 assembly implementation of 8-bit mono audio sample mixing. Processes 256 samples per invocation, applying volume translation and harsh clipping to prevent distortion during real-time audio mixing.

## Core Responsibilities
- Fast inner-loop mixing of 8-bit mono audio samples
- Applies volume translation via lookup table
- Applies harsh clipping via lookup table to prevent overflow
- Handles fractional sample position tracking for potential resampling
- Uses self-modifying code for runtime table pointer inlining
- Processes samples in pairs (two per iteration) for throughput optimization

## Key Types / Data Structures
None (tables are referenced externally; only scalars used in computation).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| apatch1, apatch2 | immediate value (self-modifying) | static | Volume table pointer inlined into mov instructions |
| apatch3, apatch4 | immediate value (self-modifying) | static | Harsh clip table pointer inlined into mov instructions |
| apatch5, apatch6 | immediate value (self-modifying) | static | Sample rate/increment inlined into add instructions |
| MixBufferSize | constant | static | 256 samples per mix pass |

## Key Functions / Methods

### MV_Mix8BitMonoFast_
- **Signature:**  
  `void MV_Mix8BitMonoFast_(uint32_t position[eax], uint32_t rate[edx], uint8_t *volume_table[ecx], uint8_t *harsh_clip_table[ebx], uint8_t *dest[edi], uint8_t *src[esi])`

- **Purpose:**  
  Mix 256 8-bit mono samples from source into destination buffer, applying per-sample volume and clipping.

- **Inputs:**
  - `eax`: fractional sample position (high 16 bits = integer offset, low 16 bits = fraction)
  - `edx`: sampling rate increment (fixed-point)
  - `ecx`: pointer to 256-entry volume translation table
  - `ebx`: pointer to harsh clip table (offset by 128 internally)
  - `edi`: destination buffer pointer
  - `esi`: source buffer pointer

- **Outputs/Return:**  
  Mixed samples written in-place to destination buffer; position/rate not returned (assumed caller-managed).

- **Side effects:**  
  - Writes 256 bytes to `edi` (destination buffer)
  - Self-modifies instructions at `apatch1–apatch6` to inline table pointers
  - Pushes/pops `ebp` for stack frame

- **Calls:**  
  None (direct).

- **Notes:**  
  - **Self-modifying code**: Patches immediate operands at runtime to avoid register pressure; common in '90s audio DSP code.
  - **Fractional position tracking**: High 16 bits used as offset, low 16 bits tracked as fractional part; allows smooth pitch shifting.
  - **Dual-sample pipeline**: Processes two samples per loop iteration to hide memory latency.
  - **Harsh clipping table offset**: `ebx` is offset by 128 before patching, suggesting the clip table is symmetrical around index 128.

## Control Flow Notes
1. **Setup phase**: Self-modifies code at six patch points to inline table pointers and rate.
2. **Main loop** (`mix8Mloop`): Iterates 128 times (256 samples ÷ 2).
   - Load sample pair from source using fractional position.
   - Volume-translate each sample.
   - Mix (add) into current destination samples.
   - Harsh-clip result.
   - Write back to destination; advance position by rate.
3. **Cleanup**: Pop `ebp`, return.

Fits into the audio frame/buffer processing pipeline (likely called once per audio frame to mix one sample buffer).

## External Dependencies
- **Includes**: None visible (pure x86 assembly).
- **External symbols**: Invoked as a callable function (likely from C/C++ audio mixer); table pointers and buffers supplied by caller.
- **Assembler**: TASM (Turbo Assembler) syntax (`.386`, `.MODEL flat`, `SEGMENT`, `PROC`/`ENDP`).
