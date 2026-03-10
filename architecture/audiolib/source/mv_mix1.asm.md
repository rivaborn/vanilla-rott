# audiolib/source/mv_mix1.asm

## File Purpose
Low-level optimized x86 assembly mixer for 8-bit mono audio samples. Uses self-modifying code and lookup tables to rapidly apply volume scaling and clipping during audio buffer mixing.

## Core Responsibilities
- Mix 8-bit mono audio samples with fractional sample positioning
- Apply volume transformation via lookup table
- Apply harsh clipping via lookup table to prevent distortion
- Self-patch code with runtime parameters for zero-overhead parameter passing
- Process fixed buffer size (256 samples) per invocation

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| apatch1–apatch6 | Code address offsets | Static | Runtime-patched instruction operands for rate, volume table, and clip table parameters |
| MixBufferSize | Constant (256) | Static | Number of samples processed per function call |

## Key Functions / Methods

### MV_Mix8BitMonoFast_
- **Signature:** `void MV_Mix8BitMonoFast_()`  
  Parameters passed in registers:
  - `eax`: position (fractional sample index, upper 16 bits = integer offset)
  - `edx`: rate (sample rate advancement per iteration)
  - `ecx`: volume table pointer
  - `ebx`: harsh clip table pointer (pre-adjusted by +128)
  - `edi`: destination buffer pointer
  - `esi`: source buffer pointer

- **Purpose:**  
  Core inner loop that mixes one audio buffer of 8-bit mono samples with volume scaling and clipping.

- **Inputs:**  
  - Registers: position, rate, volume table, clip table, source/destination pointers
  - Source buffer: indexed via fractional position
  - Lookup tables: volume and clipping

- **Outputs/Return:**  
  Modifies destination buffer in-place; fractional position advanced but not returned.

- **Side effects:**  
  - Modifies destination buffer (I/O)
  - Self-modifies code at apatch1, apatch2, apatch3 locations (inline parameter patching)
  - Advances through source buffer

- **Calls:**  
  None (pure computation with table lookups)

- **Notes:**  
  - Fractional position uses fixed-point: upper 16 bits = integer sample index
  - Harsh clip table is offset by +128 before being patched, enabling signed indexing
  - Self-modifying code (1990s–2000s optimization): avoids register/memory loads in hot loop by patching literals
  - Processes exactly 256 samples per call

## Control Flow Notes
Entry point for audio frame mixing. Assumes called repeatedly during audio buffer rendering. Fractional positioning enables pitch shifting and resampling by varying the rate parameter.

## External Dependencies
None (standalone assembly).
