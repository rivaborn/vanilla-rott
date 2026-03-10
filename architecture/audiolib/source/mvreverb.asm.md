# audiolib/source/mvreverb.asm

## File Purpose
Implements audio reverb/echo effects processing routines in x86 assembly for the audio mixing engine. Provides both table-lookup based reverb (with volume attenuation) and shift-based fast reverb operations for 8-bit and 16-bit PCM audio samples.

## Core Responsibilities
- Process 16-bit and 8-bit audio samples through reverb effects
- Apply volume table transformations to attenuate/transform sample data
- Implement fast reverb path using arithmetic right shifts with rounding
- Copy and mix audio samples from source to destination buffers
- Support self-modifying code for runtime shift amount specialization

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rpatch16` | Label | Static | Location in `MV_16BitReverbFast_` where shift amount is self-patched at runtime |
| `rpatch8` | Label | Static | Location in `MV_8BitReverbFast_` where shift amount is self-patched at runtime |

## Key Functions / Methods

### MV_16BitReverb_
- **Signature:** `(eax=src_addr, edx=dst_addr, ebx=volume_table, ecx=sample_count) → void`
- **Purpose:** Apply 16-bit reverb with per-byte volume table lookup
- **Inputs:** eax=source buffer address, edx=destination buffer address, ebx=volume lookup table base, ecx=number of samples
- **Outputs/Return:** Samples written to destination; no return value
- **Side effects:** Modifies esi, edi, eax, edx, ecx; writes to destination buffer
- **Calls:** None (tight loop structure)
- **Notes:** Treats 16-bit sample as two 8-bit values; indexes volume table separately for high/low bytes using `2*byte_value+table_base`. Adds 0x80 bias to mixed result.

### MV_8BitReverb_
- **Signature:** `(eax=src_addr, edx=dst_addr, ebx=volume_table, ecx=sample_count) → void`
- **Purpose:** Apply 8-bit reverb with volume table lookup
- **Inputs:** eax=source address, edx=destination address, ebx=volume table, ecx=sample count
- **Outputs/Return:** Samples written to destination
- **Side effects:** Modifies al, esi, edi, ecx; writes to destination buffer
- **Calls:** None
- **Notes:** Simpler path than 16-bit; single byte per sample. Adds 0x80 bias to transformed sample.

### MV_16BitReverbFast_
- **Signature:** `(eax=src_addr, edx=dst_addr, ebx=sample_count, ecx=shift_amount) → void`
- **Purpose:** Apply 16-bit reverb using arithmetic right shift (self-modifying fast path)
- **Inputs:** eax=source address, edx=destination address, ebx=sample count, ecx=shift amount
- **Outputs/Return:** Samples written to destination
- **Side effects:** Self-modifying code—patches instruction at `rpatch16` label with shift value from cl; writes samples to destination
- **Calls:** None
- **Notes:** Uses runtime code patching to specialize the shift instruction. The `rpatch16:` location receives the shift amount, allowing a single compiled loop to work with different reverb strengths.

### MV_8BitReverbFast_
- **Signature:** `(eax=src_addr, edx=dst_addr, ebx=sample_count, ecx=shift_amount) → void`
- **Purpose:** Apply 8-bit reverb using arithmetic right shift with sign-aware rounding
- **Inputs:** eax=source address, edx=destination address, ebx=sample count, ecx=shift amount
- **Outputs/Return:** Samples written to destination
- **Side effects:** Self-modifying code—patches shift at `rpatch8` label; pre-computes rounding offset in edx; writes samples to destination
- **Calls:** None
- **Notes:** Complex logic: pre-computes an offset (edx = 0x80 − (0x80 >> shift)) to handle rounding. Flips sign bit, shifts, then adds sign bit back to round toward zero. Demonstrates optimization for sign-aware arithmetic.

## Control Flow Notes
These are frame-oriented audio processing routines, likely called during audio mixer updates. They iterate over sample buffers in tight loops. The "fast" variants (`MV_16BitReverbFast_`, `MV_8BitReverbFast_`) trade code size/flexibility for speed by using self-modifying code to specialize the shift amount at runtime, avoiding a register load inside the inner loop.

## External Dependencies
- None: pure x86 assembly with no external symbols or imports.
