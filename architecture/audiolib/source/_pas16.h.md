# audiolib/source/_pas16.h

## File Purpose
Private header for PAS16.C that defines constants, macros, and function declarations for ProAudio Spectrum 16 (PAS16) soundcard driver support. Provides hardware register addresses, bit flags, audio format definitions, and driver interface structures for retro-era audio hardware integration.

## Core Responsibilities
- Define hardware I/O port addresses and register layouts for PAS16 card
- Provide bit flag constants for audio modes (mono/stereo, 8-bit/16-bit) and hardware control
- Declare low-level register access functions (`PAS_Read`, `PAS_Write`)
- Declare card initialization and discovery functions (`PAS_FindCard`, `PAS_CheckForDriver`, `PAS_GetCardSettings`)
- Declare DMA and interrupt control functions (`PAS_SetupDMABuffer`, `PAS_EnableInterrupt`, `PAS_ServiceInterrupt`)
- Define structures for mapping hardware state (`MVState`) and function pointers (`MVFunc`)
- Provide macros for sample rate calculation and audio format encoding

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MVState` | struct | Maps all PAS16 hardware registers to named fields (FM synth, PCM, MIDI, mixer, timers) |
| `MVFunc` | struct | Holds function pointers to card driver operations (SetMixer, SetVolume, ReadSound, etc.) |

## Global / File-Static State
None.

## Key Functions / Methods

### PAS_CheckForDriver
- Signature: `int PAS_CheckForDriver(void)`
- Purpose: Verify if PAS16 driver is installed/accessible
- Inputs: None
- Outputs/Return: Non-zero if driver found
- Side effects: Likely probes hardware or checks interrupt vectors
- Calls: Not inferable from this file

### PAS_FindCard
- Signature: `int PAS_FindCard(void)`
- Purpose: Locate PAS16 card on system bus
- Inputs: None
- Outputs/Return: Non-zero if card found
- Side effects: Iterates through possible base addresses (DEFAULT_BASE, ALT_BASE_1-3)
- Calls: `PAS_TestAddress`

### PAS_GetStateTable / PAS_GetFunctionTable
- Signature: `MVState *PAS_GetStateTable(void)`, `MVFunc *PAS_GetFunctionTable(void)`
- Purpose: Return pointers to card state and function lookup structures
- Inputs: None
- Outputs/Return: Pointer to persistent structures
- Side effects: None
- Calls: Not inferable from this file

### PAS_SetupDMABuffer
- Signature: `int PAS_SetupDMABuffer(char *BufferPtr, int BufferSize, int mode)`
- Purpose: Configure DMA transfer buffer for audio playback/recording
- Inputs: Buffer pointer, size, mode (RECORD or PLAYBACK)
- Outputs/Return: Status code
- Side effects: Configures card DMA engine
- Calls: Not inferable from this file

### PAS_ServiceInterrupt
- Signature: `void interrupt far PAS_ServiceInterrupt(void)`
- Purpose: Hardware interrupt handler for PAS16 card events
- Inputs: None (CPU interrupt context)
- Outputs/Return: None
- Side effects: Reads interrupt status, clears interrupt, updates audio buffers
- Calls: Not inferable from this file
- Notes: `far` keyword indicates real-mode x86 interrupt handler; marked as interrupt routine

### PAS_TestAddress
- Signature: `int PAS_TestAddress(int address)`
- Purpose: Test if a hardware register address is valid (inline assembly implementation)
- Inputs: Address to test in `eax` register
- Outputs/Return: Register revision bits if valid (0xff indicates invalid)
- Side effects: I/O port access (reads/writes register 0xb8b)
- Calls: Hardware-level (no function calls)
- Notes: Implemented via `#pragma aux` inline assembly; verifies address by XORing with revision bits

## Control Flow Notes
This header supports initialization, interrupt-driven audio processing, and shutdown phases:
- **Init**: `PAS_CheckForDriver` → `PAS_FindCard` → `PAS_GetCardSettings` → `PAS_GetStateTable`/`PAS_GetFunctionTable`
- **Audio setup**: `PAS_SetupDMABuffer` → `PAS_BeginTransfer` → register sample rate/size via `PAS_Write`
- **Runtime**: `PAS_ServiceInterrupt` (triggered by hardware on sample completion)
- **Shutdown**: `PAS_SaveState` / `PAS_RestoreState` for state persistence

## External Dependencies
- **No includes** (this is a private header; implementation in PAS16.C)
- **Assumes**: Watcom C/C++ compiler (uses `#pragma aux` for inline assembly and `far` keyword)
- **Real-mode x86 assumption**: Code targets 16-bit x86 DOS/real-mode environment (interrupt handlers, I/O port access, far pointers)
- **Symbols defined elsewhere**: All function implementations in PAS16.C; structures filled by driver calls
