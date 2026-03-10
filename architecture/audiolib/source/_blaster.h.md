# audiolib/source/_blaster.h

## File Purpose
Private header defining Sound Blaster audio hardware constants, port addresses, DSP command codes, and mixer register definitions. Supports legacy SoundBlaster cards (1xx–4xx versions) for DOS-era audio playback and recording.

## Core Responsibilities
- Define I/O port offsets for Sound Blaster hardware communication (reset, read, write, data-available ports)
- Define mixer register addresses and control bits for audio configuration
- Define DSP (Digital Signal Processor) command opcodes for audio operations
- Provide helper macros for audio sampling rate and time-constant calculations
- Define format flags for audio data (signed/unsigned, mono/stereo)
- Parse BLASTER environment variable tokens (address, interrupt, DMA channels, card type)
- Define card capability tracking structure

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CARD_CAPABILITY | struct | Stores audio card capabilities (mixer support, sampling rate range) |

## Global / File-Static State
None.

## Key Functions / Methods
None (header-only definitions).

## Macro Utilities

### CalcTimeConstant
- **Formula:** `(65536L - (256000000L / ((samplesize) * (rate)))) >> 8`
- **Purpose:** Convert sampling rate and sample size to DSP time constant
- **Inputs:** `rate` (sampling frequency), `samplesize` (bits per sample)
- **Outputs:** DSP time constant value

### CalcSamplingRate
- **Formula:** `256000000L / (65536L - (tc << 8))`
- **Purpose:** Inverse calculation—derive sampling rate from DSP time constant
- **Inputs:** `tc` (DSP time constant)
- **Outputs:** Sampling rate (Hz)

### lobyte / hibyte
- **Purpose:** Extract low/high byte from integer
- **Inputs:** Numeric value
- **Outputs:** Low or high byte as int

## Control Flow Notes
Compile-time header only. Defines hardware constants and types consumed by `blaster.c` and other audio library modules at initialization and during audio I/O operations.

## External Dependencies
- None (self-contained hardware definitions)
