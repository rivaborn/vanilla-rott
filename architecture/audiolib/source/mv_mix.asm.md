# audiolib/source/mv_mix.asm

## File Purpose
Low-level x86-32 assembly implementation of audio sample mixing routines. Provides four functions to mix audio at different bit depths (8-bit, 16-bit) and channel configurations (mono, stereo), using self-modifying code for runtime parameter injection.

## Core Responsibilities
- Resample audio from source buffer using fractional position tracking and rate scaling
- Apply per-channel volume translation via lookup tables
- Accumulate (mix) resampled samples with destination output buffer
- Clip results to valid audio ranges (harsh clipping for 8-bit via table, explicit bounds-checking for 16-bit)
- Update global mixing state (destination write position and playback position) after each frame
- Use self-modifying code patching to inject runtime parameters (volume tables, sample offsets, rates) into inner loops

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_MV_HarshClipTable` | DWORD | global (EXTRN) | Lookup table for clipping 8-bit samples (offset by 128 for signed range) |
| `_MV_MixDestination` | DWORD | global (EXTRN) | Current write position in output buffer |
| `_MV_MixPosition` | DWORD | global (EXTRN) | Fractional playback position (22.10 fixed-point or similar) |
| `_MV_LeftVolume` | DWORD | global (EXTRN) | Left channel volume lookup table pointer |
| `_MV_RightVolume` | DWORD | global (EXTRN) | Right channel volume lookup table pointer |
| `_MV_SampleSize` | DWORD | global (EXTRN) | Size of samples in source buffer (1 or 2 bytes per sample) |
| `_MV_RightChannelOffset` | DWORD | global (EXTRN) | Byte offset to right channel in stereo interleaved buffer |

## Key Functions / Methods

### MV_Mix8BitMono_
- **Signature:** `void MV_Mix8BitMono_(void)` (via registers: `eax`=position, `edx`=rate, `ebx`=source_start, `ecx`=sample_count)
- **Purpose:** Mix 8-bit mono audio at variable playback rate, processing two samples per iteration.
- **Inputs:**  
  - `eax`: fractional playback position  
  - `edx`: rate increment per sample  
  - `ebx`: source buffer start address  
  - `ecx`: number of samples to mix
- **Outputs/Return:** None (updates globals `_MV_MixDestination`, `_MV_MixPosition`)
- **Side effects:** 
  - Self-modifying code: patches volume table pointers (`apatch1`, `apatch2`), harsh clip table (`apatch3`, `apatch4`), rate increment (`apatch5`, `apatch6`), and destination increment size (`apatch7`, `apatch8`, `apatch9`) into instruction operands
  - Reads from source buffer and destination buffer  
  - Writes mixed/clipped samples to destination buffer  
  - Updates global mix position and destination pointer
- **Calls:** None (self-contained inner loop)
- **Notes:** 
  - Processes samples in pairs for throughput; divides sample count by 2 (`shr ecx, 1`)
  - Fractional position is right-shifted by 16 to get sample index  
  - Volume lookup uses 2-byte table entries (`2*eax`); harsh clipping uses table lookup  
  - Loop exits if sample count is 0

### MV_Mix8BitStereo_
- **Signature:** `void MV_Mix8BitStereo_(void)` (via registers: `eax`=position, `edx`=rate, `ebx`=source_start, `ecx`=sample_count)
- **Purpose:** Mix 8-bit stereo audio with separate left and right volume tables.
- **Inputs:**  
  - `eax`: fractional playback position  
  - `edx`: rate increment  
  - `ebx`: source buffer start  
  - `ecx`: number of samples
- **Outputs/Return:** None (updates `_MV_MixDestination`, `_MV_MixPosition`)
- **Side effects:** 
  - Self-modifying code patches for left/right volume tables (`bpatch1`, `bpatch2`), rate (`bpatch3`), harsh clip tables (`bpatch4`, `bpatch5`), right channel offset (`bpatch6`, `bpatch7`), and destination increment (`bpatch8`)
  - Reads stereo interleaved samples from source  
  - Writes left and right samples to destination with per-channel clipping
- **Calls:** None
- **Notes:** 
  - Processes one stereo sample (L+R) per iteration  
  - Right channel sample offset is patched per-call via `_MV_RightChannelOffset`  
  - Mono source position is used; left and right are derived from same sample index

### MV_Mix16BitMono_
- **Signature:** `void MV_Mix16BitMono_(void)` (via registers: `eax`=position, `edx`=rate, `ebx`=source_start, `ecx`=sample_count)
- **Purpose:** Mix 16-bit signed mono audio with explicit clipping to ±32768.
- **Inputs:**  
  - `eax`: fractional playback position  
  - `edx`: rate increment  
  - `ebx`: source start  
  - `ecx`: sample count
- **Outputs/Return:** None (updates `_MV_MixDestination`, `_MV_MixPosition`)
- **Side effects:** 
  - Self-modifying code patches volume tables (`cpatch1`, `cpatch2`), rate increments (`cpatch3`, `cpatch4`), and sample size (`cpatch5`, `cpatch6`, `cpatch7`)
  - Reads word (16-bit) samples; writes clipped results  
  - Sample size is doubled for 16-bit (`add bl, bl`)
- **Calls:** None
- **Notes:** 
  - Processes two 16-bit samples per iteration (`shr ecx, 1`)  
  - Clipping uses explicit comparisons (jump on `>= -32768` and `<= 32767`) rather than table lookup  
  - Label pairs (`m16skip1/2`, `m16skip3/4`) implement clipping conditional jumps

### MV_Mix16BitStereo_
- **Signature:** `void MV_Mix16BitStereo_(void)` (via registers: `eax`=position, `edx`=rate, `ebx`=source_start, `ecx`=sample_count)
- **Purpose:** Mix 16-bit signed stereo audio with per-channel volume and explicit clipping.
- **Inputs:**  
  - `eax`: fractional playback position  
  - `edx`: rate increment  
  - `ebx`: source start  
  - `ecx`: sample count
- **Outputs/Return:** None (updates `_MV_MixDestination`, `_MV_MixPosition`)
- **Side effects:** 
  - Self-modifying code patches left/right volume (`dpatch1`, `dpatch2`), rate (`dpatch3`), right channel offset (`dpatch4`, `dpatch5`), and destination increment (`dpatch6`)  
  - Reads stereo word samples; writes clipped L/R to destination
- **Calls:** None
- **Notes:** 
  - Processes one stereo sample pair per iteration  
  - Clipping uses same pattern as 16-bit mono (`s16skip1–4` labels)  
  - Right channel offset injected at `dpatch4` and `dpatch5`

## Control Flow Notes
These functions are mixing/resampling engine entry points, typically called once per audio frame. **Initialization phase** (common to all four): parameters are read from globals and self-modifying code patches are applied to instruction operands (volume tables, clip tables, rate scales, offsets). **Main loop phase**: samples are fetched from source using interpolated (fractional) indices, volume-adjusted, mixed with destination, clipped, and written. **Exit phase**: updated mix position and destination pointer are written back to globals. The functions are frame-driven (not interrupt-driven) and assume the caller manages buffer boundaries.

## External Dependencies
- **External symbols (EXTRN):**
  - `_MV_HarshClipTable` — 8-bit clipping lookup table (offset ±128 for signed range)
  - `_MV_MixDestination`, `_MV_MixPosition` — mixing state variables
  - `_MV_LeftVolume`, `_MV_RightVolume` — channel volume lookup tables
  - `_MV_SampleSize`, `_MV_RightChannelOffset` — format configuration constants
- **Assembler directives:** `IDEAL` mode (Borland TASM), `p386` (32-bit), `MODEL flat` (flat memory), `MASM` compatibility
- **No includes or inter-file function calls** — self-contained inner loops with code patching
