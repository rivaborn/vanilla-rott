# audiolib/source/mv_mix16.asm

## File Purpose
Low-level x86 assembly implementation of audio sample mixing for a game engine. Provides optimized mixing routines that combine source audio into a destination buffer with volume scaling, sample rate conversion, and overflow clipping. Supports 8-bit and 16-bit samples in mono and stereo configurations.

## Core Responsibilities
- Mix audio samples from source buffer into destination buffer with volume translation
- Apply per-channel volume via lookup table (8-bit) or direct scaling (16-bit)
- Perform sample rate conversion using 16.16 fixed-point fractional position arithmetic
- Prevent overflow via harsh clipping (lookup table for 8-bit, direct comparison for 16-bit)
- Support dual sample processing per loop iteration for efficiency
- Use runtime self-modifying code ("patching") to inject configuration values into inner loops

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_MV_HarshClipTable` | DWORD | global | Clipping lookup table (offset +128) for 8-bit sample overflow protection |
| `_MV_MixDestination` | DWORD | global | Current write position in output mix buffer |
| `_MV_MixPosition` | DWORD | global | Current playback position (16.16 fixed-point) |
| `_MV_LeftVolume` | DWORD | global | Pointer to left channel volume translation table |
| `_MV_RightVolume` | DWORD | global | Pointer to right channel volume translation table |
| `_MV_SampleSize` | DWORD | global | Sample stride in bytes (for skip calculations) |
| `_MV_RightChannelOffset` | DWORD | global | Byte offset from left to right channel in stereo |

## Key Functions / Methods

### MV_Mix8BitMono16_
- **Signature:** `void MV_Mix8BitMono16_()`
  - **Input registers:** `eax` = position (16.16 fixed), `edx` = rate delta, `ebx` = source buffer start, `ecx` = sample count
  - **Output:** Updates `_MV_MixDestination` and `_MV_MixPosition` globals
- **Purpose:** Mix 8-bit mono samples into destination at 16-bit precision with volume scaling and clipping
- **Inputs:** Four arguments passed in x86 registers (position, rate, start address, count)
- **Outputs/Return:** Modifies destination buffer in-place; updates position/destination globals; returns void
- **Side effects:** Writes to `_MV_MixDestination` and `_MV_MixPosition`; modifies destination buffer; self-modifies code at `apatch1`–`apatch9` labels
- **Calls:** None (pure assembly, no external calls)
- **Notes:** Processes **two samples per loop iteration** for efficiency. Uses self-modifying code to inject: volume table pointer (`apatch1`, `apatch2`), harsh clip table (`apatch3`, `apatch4`), rate delta (`apatch5`, `apatch6`), and sample size (`apatch7`, `apatch8`, `apatch9`). Uses XOR with 0x80 to convert sample bias. Loop condition checks `ecx` after halving sample count.

### MV_Mix8BitStereo16_
- **Signature:** `void MV_Mix8BitStereo16_()`
  - **Input registers:** `eax` = position, `edx` = rate, `ebx` = source start, `ecx` = sample count
  - **Output:** Updates `_MV_MixDestination` and `_MV_MixPosition` globals
- **Purpose:** Mix 8-bit stereo samples with separate left/right volume scaling
- **Inputs:** Position, rate, source address, sample count
- **Outputs/Return:** Modifies destination buffer; updates globals
- **Side effects:** Writes to globals; modifies code at `bpatch1`–`bpatch8`
- **Calls:** None
- **Notes:** Processes **one stereo sample pair per iteration**. Injects separate left (`bpatch1`) and right (`bpatch2`) volume tables. Right channel offset (`bpatch6`, `bpatch7`) allows stereo interleaving. Harsh clip table (`bpatch4`, `bpatch5`). Sample size patch at `bpatch8`.

### MV_Mix16BitMono16_
- **Signature:** `void MV_Mix16BitMono16_()`
  - **Input registers:** `eax` = position, `edx` = rate, `ebx` = source, `ecx` = count
  - **Output:** Updates `_MV_MixDestination` and `_MV_MixPosition`
- **Purpose:** Mix 16-bit signed mono samples with volume scaling and signed clipping
- **Inputs:** Position, rate, source buffer, sample count
- **Outputs/Return:** Modifies destination; updates globals
- **Side effects:** Writes to globals; patches code at `cpatch1`–`cpatch5`
- **Calls:** None
- **Notes:** Processes **one 16-bit sample per iteration**. Uses **direct clipping comparisons** (`cmp eax, -32768` / `cmp eax, 32767`) instead of lookup table; more efficient for 16-bit range. Extracts high and low bytes for volume translation via lookup table. XOR with 0x8000 to flip signed/unsigned representation.

### MV_Mix16BitStereo16_
- **Signature:** `void MV_Mix16BitStereo16_()`
  - **Input registers:** `eax` = position, `edx` = rate, `ebx` = source, `ecx` = count
  - **Output:** Updates globals
- **Purpose:** Mix 16-bit signed stereo samples with separate left/right volume and clipping
- **Inputs:** Position, rate, source address, sample count
- **Outputs/Return:** Modifies destination; updates globals
- **Side effects:** Patches code at `dpatch1`–`dpatch9`; writes to globals and destination
- **Calls:** None
- **Notes:** Processes **one stereo sample pair per iteration**. Injects separate volume tables for left (`dpatch1`, `dpatch2`) and right (`dpatch3`, `dpatch4`). Right channel offset patched at `dpatch7`, `dpatch8`. Sample size at `dpatch9`. Source buffer pointer self-injected at `dpatch6`. Applies signed clipping separately to left (s16skip1/2) and right (s16skip3/4) samples.

## Control Flow Notes
All four functions follow the same initialization→loop→cleanup pattern:
1. **Setup phase (pushad):** Save registers; copy runtime parameters from globals into self-modifying code patches (apatch/bpatch/cpatch/dpatch labels)
2. **Loop entry:** Initialize first sample calculation (16.16 fixed-point shift and lookup)
3. **Main loop:** Fetch sample → apply volume table → mix with destination → apply clip → write result → advance position → loop
4. **Cleanup (popad):** Restore all registers and return

Position is tracked in **16.16 fixed-point format** (ebp register), advanced by rate delta per sample. Destination pointer (edi) increments by 2 (8-bit) or 4 (16-bit) per sample iteration.

## External Dependencies
- **TASM directives:** IDEAL, MODEL flat, MASM, ALIGN, PROC, PUBLIC, EXTRN, ENDS, END
- **External symbols:** `_MV_HarshClipTable`, `_MV_MixDestination`, `_MV_MixPosition`, `_MV_LeftVolume`, `_MV_RightVolume`, `_MV_SampleSize`, `_MV_RightChannelOffset` (all defined elsewhere, likely in C runtime)
- **No internal calls:** Pure assembly with no dependencies on other functions in this file
