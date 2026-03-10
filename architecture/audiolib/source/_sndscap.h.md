# audiolib/source/_sndscap.h

## File Purpose
Private header for low-level Ensoniq Soundscape sound card driver implementation. Defines hardware register offsets, firmware commands, audio format constants, and function declarations for direct hardware manipulation including interrupt handling, DMA control, and codec configuration.

## Core Responsibilities
- Ensoniq gate-array (ODIE/OPUS/MiMIC) hardware register offsets and indirect register addresses
- AD-1848 audio codec register definitions and indirect register layout
- Audio format mode flags (mono/stereo, 8/16-bit) and sample size macros
- Firmware command codes and status bit masks for device communication
- x86 interrupt controller definitions and IRQ handling setup
- Function declarations for gate-array I/O, interrupt management, DMA configuration, and device initialization

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### SOUNDSCAPE_ServiceInterrupt
- Signature: `static void __interrupt __far SOUNDSCAPE_ServiceInterrupt( void )`
- Purpose: Hardware interrupt service routine for handling Soundscape sound card interrupts
- Inputs: None (invoked by CPU on interrupt)
- Outputs/Return: None (void ISR)
- Side effects: Handles hardware interrupt, coordinates with interrupt controllers
- Calls: Not inferable from this file
- Notes: Declared with `__interrupt __far` keywords (DOS/16-bit convention); must save/restore registers

### SOUNDSCAPE_EnableInterrupt / SOUNDSCAPE_DisableInterrupt
- Signature: `static void SOUNDSCAPE_EnableInterrupt(void)` / `static void SOUNDSCAPE_DisableInterrupt(void)`
- Purpose: Manage CPU interrupt enable/disable state for Soundscape hardware operations
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies CPU interrupt mask register via interrupt controller ports
- Calls: Not inferable from this file
- Notes: Critical for thread-safe hardware access in ISR context

### ga_read / ga_write
- Signature: `static int ga_read(int rnum)` / `static void ga_write(int rnum, int value)`
- Purpose: Read/write indirect register on Ensoniq gate-array chip
- Inputs: `rnum` = register number; `value` = data to write
- Outputs/Return: `ga_read` returns register value; `ga_write` returns void
- Side effects: Direct I/O port access via `GA_REGADDR` and `GA_REGDATA`
- Calls: Not inferable from this file
- Notes: Implements indirect addressing pattern common to gate-array chips

### ad_read / ad_write
- Signature: `static int ad_read(int rnum)` / `static void ad_write(int rnum, int value)`
- Purpose: Read/write indirect register on AD-1848 audio codec
- Inputs: `rnum` = register number; `value` = data to write
- Outputs/Return: `ad_read` returns register value; `ad_write` returns void
- Side effects: Direct I/O port access via `AD_REGADDR` and `AD_REGDATA`
- Calls: Not inferable from this file
- Notes: Codec-specific register interface; similar indirect pattern to gate-array

### SOUNDSCAPE_SetupDMABuffer
- Signature: `static int SOUNDSCAPE_SetupDMABuffer(char *BufferPtr, int BufferSize, int mode)`
- Purpose: Configure DMA buffer for PCM audio playback
- Inputs: `BufferPtr` = audio sample buffer; `BufferSize` = buffer size in bytes; `mode` = audio format mode flags
- Outputs/Return: Integer status (success/failure)
- Side effects: Locks memory, configures gate-array DMA registers
- Calls: Not inferable from this file
- Notes: Memory locking required for DMA access in protected mode

### SOUNDSCAPE_BeginPlayback
- Signature: `static int SOUNDSCAPE_BeginPlayback(int length)`
- Purpose: Start audio playback from configured DMA buffer
- Inputs: `length` = playback length in samples or bytes
- Outputs/Return: Integer status
- Side effects: Starts DMA controller, enables audio output
- Calls: Not inferable from this file

### SOUNDSCAPE_LockMemory / SOUNDSCAPE_UnlockMemory
- Signature: `static int SOUNDSCAPE_LockMemory(void)` / `static void SOUNDSCAPE_UnlockMemory(void)`
- Purpose: Lock/unlock DMA buffer in physical memory for DMA device access
- Inputs: None
- Outputs/Return: `LockMemory` returns status; `UnlockMemory` returns void
- Side effects: Interacts with memory manager (DOS/Windows DPMI)
- Calls: Not inferable from this file
- Notes: Required for DMA in real/protected mode; prevents memory swapping

### SOUNDSCAPE_FindCard / SOUNDSCAPE_Setup
- Signature: `static int SOUNDSCAPE_FindCard(void)` / `static int SOUNDSCAPE_Setup(void)`
- Purpose: Locate Soundscape hardware and initialize device configuration
- Inputs: None
- Outputs/Return: Integer status/handle
- Side effects: Hardware detection, gate-array and codec initialization
- Calls: Not inferable from this file
- Notes: Entry points for driver initialization

**Helper functions** (trivial): `tdelay()` (timing delay), `pcm_format()` (audio format setup), `parse()` (configuration string parsing), `allocateTimerStack()` / `deallocateTimerStack()` (stack management for interrupt handlers).

## Control Flow Notes
This header supports initialization phase (FindCard → Setup → LockMemory → SetupDMABuffer) followed by frame/playback loop (BeginPlayback → ServiceInterrupt handles DMA completion) and shutdown (UnlockMemory). Interrupt-driven model: ServiceInterrupt() responds to hardware completion signals, coordinated by EnableInterrupt/DisableInterrupt gates.

## External Dependencies
- **Standard C**: `FILE` type (for configuration parsing)
- **x86 Hardware**: Interrupt controller I/O ports (0x20, 0x21, 0xa0, 0xa1); DMA controller; port I/O via `__interrupt` pragma
- **DOS/DPMI Memory Model**: Protected-mode memory locking, far pointers, interrupt vector installation
- **External symbols**: Audio buffer management, hardware base address (not defined here)
