# audiolib/source/_blaster.h — Enhanced Analysis

## Architectural Role

This private header is the **hardware interface specification layer** for the Sound Blaster audio subsystem. It translates physical hardware port addresses and DSP opcodes into named constants consumed by the public Blaster driver API (`blaster.c` and `blastold.c`). The file acts as the central registry of hardware contracts—without it, the entire Sound Blaster playback and recording pipeline (`BLASTER_BeginBufferedPlayback`, `BLASTER_SetMixMode`, `BLASTER_ServiceInterrupt`, etc.) would have raw magic numbers scattered throughout the driver code.

## Key Cross-References

### Incoming (Direct Consumers)
- **blaster.c** — Primary driver implementation; directly uses all DSP command codes, port addresses, mixer registers, and the `CalcTimeConstant`/`CalcSamplingRate` macros for audio rate negotiation
- **blastold.c** — Legacy compatibility driver; reuses identical port and DSP constants for backward-compatible hardware support
- Functions that consume these constants:
  - `BLASTER_WriteDSP` / `BLASTER_ReadDSP` — directly write/read DSP opcodes (e.g., `DSP_8BitDAC`, `DSP_16BitDAC`)
  - `BLASTER_ServiceInterrupt` — reads `BLASTER_DataAvailablePort` and services DMA interrupts
  - `BLASTER_SetMixMode` — writes to `BLASTER_MixerAddressPort` / `BLASTER_MixerDataPort` with mixer register addresses
  - `BLASTER_SetPlaybackRate` — uses `CalcTimeConstant` to convert Hz into DSP time constants
  - `BLASTER_Init` / `BLASTER_ResetDSP` — use `BLASTER_ResetPort` and `DSP_Reset` opcode

### Outgoing (Dependencies)
- **None** — this is a leaf header with no external dependencies; pure constant definitions

## Design Patterns & Rationale

**Hardware Abstraction via Macros:**
- Port offsets (`0x04`, `0x0A`, etc.) and DSP opcodes are wrapped in named constants rather than scattered as magic numbers. This follows the **Hardware Abstraction Layer (HAL)** pattern, making the driver maintainable and portable across different Sound Blaster revisions.

**Variant Support via Conditional Constants:**
- Separate mixer register definitions for SB Pro (`MIXER_SBProVoice`, `MIXER_SBProMidi`) vs. SB16 (`MIXER_SB16VoiceLeft`, `MIXER_SB16VoiceRight`) reflect **compile-time or runtime hardware variant selection**. The driver likely uses `CARD_CAPABILITY` to branch on these at initialization.

**Inverse Computation Macros:**
- `CalcTimeConstant` and `CalcSamplingRate` are mathematical inverses. This encodes the DSP's hardware contract: the 256MHz internal oscillator divided by a time constant yields the sampling rate. This avoids duplicating the formula across multiple call sites.

**Legacy Defines:**
- Multiple boolean/ternary constants (`VALID`, `VALID`, `TRUE`/`FALSE`, `YES`/`NO`) suggest this codebase predates widespread use of C99 stdbool. Not reusing a single set is unusual but may reflect coding standards at Apogee circa 1994.

## Data Flow Through This File

1. **Initialization Path:**
   - `BLASTER_Init` → reads environment variable tokens (`BlasterEnv_*`) to parse address/interrupt/DMA
   - Populates `CARD_CAPABILITY` struct with detected mixer, sampling rates
   - Uses detected hardware type to branch on DSP version constants (`DSP_Version1xx`, `DSP_Version4xx`)

2. **Audio Playback Path:**
   - `BLASTER_SetPlaybackRate(Hz)` → `CalcTimeConstant(Hz, sample_size)` → DSP time constant
   - `BLASTER_SetMixMode(mode)` → writes format flags (`DSP_SignedBit`, `DSP_StereoBit`) via `BLASTER_WritePort`
   - `BLASTER_BeginBufferedPlayback` → writes `DSP_8BitDAC` or `DSP_16BitDAC` opcode
   - Interrupt fires → `BLASTER_ServiceInterrupt` reads `BLASTER_DataAvailablePort` and acknowledges via mixer

3. **State Transition:**
   - Port addresses remain constant; DSP commands transition between states (record → play, 8-bit → 16-bit)

## Learning Notes

**Era-Specific Hardware Detail:**
This file captures the Sound Blaster ISA bus architecture—fixed port I/O, indexed mixer registers, synchronous DSP command/response via polling. Modern audio (PCI, ALSA, WASAPI) abstracts these details entirely. Studying this teaches **bare-metal hardware control** as it existed in the DOS/Windows 3.1 era.

**DSP Command Semantics:**
The DSP uses "auto-init" DMA modes (e.g., `DSP_8BitAutoInitRecord`, `DSP_8BitHighSpeedAutoInitMode`) that loop transfers without CPU intervention—a technique less common in modern interrupt-driven audio, where CPU schedules buffer handoffs explicitly.

**Sampling Rate Encoding:**
The formula `256000000L / time_constant` encodes a fixed oscillator frequency. This is not obvious from the constant names alone; a developer learning this code would need domain knowledge of Sound Blaster hardware to understand why the magic number `256000000L` appears.

## Potential Issues

**Macro Safety:**
- `lobyte(num)` and `hibyte(num)` lack parentheses around `num`, risking operator precedence bugs: `hibyte(a + b)` would fail.
- `CalcTimeConstant` and `CalcSamplingRate` lack parentheses around arguments; `CalcTimeConstant(x+1, y*2)` would misbehave.

**Incomplete Variant Coverage:**
The mixer register set is incomplete—only SB Pro and SB16 are covered. Earlier SoundBlaster 1.x and early 2.x lack mixer definitions, yet the DSP version constants include `DSP_Version1xx` and `DSP_Version2xx`. If `blaster.c` attempts to set mixer registers on a 1.x card, undefined behavior may result.

**No Version Guard:**
The header lacks `#ifndef ___BLASTER_H` closing guard (shown in the file), but the constants themselves are not conditionally compiled by DSP version. Runtime checks must handle unsupported operations.
