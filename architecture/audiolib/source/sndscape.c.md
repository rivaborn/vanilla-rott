# audiolib/source/sndscape.c

## File Purpose
Low-level driver for the Ensoniq SoundScape sound card in DOS/DPMI environment. Handles hardware initialization, interrupt-driven DMA playback, and PCM audio configuration via the AD-1848 codec and gate-array chip.

## Core Responsibilities
- Detect and initialize SoundScape hardware from SNDSCAPE.INI configuration file
- Configure DMA channels and IRQ vectors for interrupt-driven audio transfer
- Manage AD-1848 codec register access (sample rate, bit depth, stereo/mono)
- Service sound card interrupts and invoke user callback at half-buffer boundaries
- Lock critical memory regions with DPMI to ensure interrupt handler stability
- Provide public API for playback control (start/stop, rate/mode configuration)
- Handle Ensoniq gate-array signal routing and chip-specific quirks (ODIE/OPUS/MMIC)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SOUNDSCAPE_Config | struct | Hardware configuration: I/O ports, DMA channel, IRQs, chip ID, routing flags |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| SOUNDSCAPE_OldInt | func ptr | static | Saved interrupt vector for restoration on shutdown |
| SOUNDSCAPE_Installed | int | static | Initialization state flag |
| SOUNDSCAPE_FoundCard | int | static | Hardware detection cache |
| SOUNDSCAPE_DMABuffer | char* | static | Start of DMA ring buffer |
| SOUNDSCAPE_DMABufferEnd | char* | static | End of DMA ring buffer |
| SOUNDSCAPE_CurrentDMABuffer | char* | static | Current playback position in buffer |
| SOUNDSCAPE_TotalDMABufferSize | int | static | Buffer size in bytes |
| SOUNDSCAPE_TransferLength | int | static | Half-buffer size (DMA chunk) |
| SOUNDSCAPE_MixMode | int | static | Format: mono/stereo, 8/16-bit |
| SOUNDSCAPE_SamplePacketSize | int | static | Bytes per sample (format-dependent) |
| SOUNDSCAPE_SampleRate | unsigned | static | Playback rate in Hz |
| SOUNDSCAPE_SoundPlaying | int | volatile/global | Active playback state (set by ISR) |
| SOUNDSCAPE_CallBack | func ptr | global | User callback invoked by interrupt handler |
| SOUNDSCAPE_IntController1Mask | int | static | Saved 8259 PIC mask register 1 |
| SOUNDSCAPE_IntController2Mask | int | static | Saved 8259 PIC mask register 2 |
| StackSelector, StackPointer | ushort/ulong | static | Interrupt handler dedicated stack |
| SOUNDSCAPE_DMAChannel | int | global | Active DMA channel number |
| SOUNDSCAPE_ErrorCode | int | global | Last error code |

## Key Functions / Methods

### SOUNDSCAPE_Init
- **Signature:** `int SOUNDSCAPE_Init(void)`
- **Purpose:** Main initialization entry point; detects card, locks memory, allocates interrupt stack, and configures hardware.
- **Inputs:** None.
- **Outputs/Return:** `SOUNDSCAPE_Ok` on success, error code on failure.
- **Side effects:** Sets `SOUNDSCAPE_Installed`, calls `SOUNDSCAPE_FindCard()`, `SOUNDSCAPE_LockMemory()`, `SOUNDSCAPE_Setup()`. Saves PIC masks. Allocates conventional memory for interrupt stack.
- **Calls:** `SOUNDSCAPE_FindCard()`, `SOUNDSCAPE_LockMemory()`, `allocateTimerStack()`, `SOUNDSCAPE_Setup()`, `SOUNDSCAPE_SetPlaybackRate()`, `SOUNDSCAPE_SetMixMode()`.
- **Notes:** Must be called before any playback. Reinitializes if already installed (calls `Shutdown` first).

### SOUNDSCAPE_Shutdown
- **Signature:** `void SOUNDSCAPE_Shutdown(void)`
- **Purpose:** Clean shutdown; halts DMA, restores interrupt vectors, frees locked memory and stack.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `SOUNDSCAPE_StopPlayback()`, disables AD-1848 interrupt, restores gate-array config, deallocates timer stack, unlocks memory, restores original interrupt vector.
- **Calls:** `SOUNDSCAPE_StopPlayback()`, `ad_write()`, `ga_write()`, `IRQ_RestoreVector()`, `deallocateTimerStack()`, `SOUNDSCAPE_UnlockMemory()`.
- **Notes:** Safe to call even if not fully initialized.

### SOUNDSCAPE_BeginBufferedPlayback
- **Signature:** `int SOUNDSCAPE_BeginBufferedPlayback(char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void (*CallBackFunc)(void))`
- **Purpose:** Start interrupt-driven playback with ring buffer and callback at half-buffer intervals.
- **Inputs:** 
  - `BufferStart`: pointer to DMA buffer
  - `BufferSize`: total buffer size in bytes
  - `NumDivisions`: number of divisions (typically 2 for double-buffer); transfer length = BufferSize / NumDivisions
  - `SampleRate`: target playback rate (normalized to 11025/22050/44100 Hz)
  - `MixMode`: format (MONO_8BIT, STEREO_8BIT, MONO_16BIT, STEREO_16BIT)
  - `CallBackFunc`: user function called after each DMA transfer
- **Outputs/Return:** `SOUNDSCAPE_Ok` or error code.
- **Side effects:** Stops any active playback, configures mix mode, sets up DMA, enables interrupts, starts AD-1848.
- **Calls:** `SOUNDSCAPE_StopPlayback()`, `SOUNDSCAPE_SetMixMode()`, `SOUNDSCAPE_SetupDMABuffer()`, `SOUNDSCAPE_SetPlaybackRate()`, `SOUNDSCAPE_SetCallBack()`, `SOUNDSCAPE_EnableInterrupt()`, `SOUNDSCAPE_BeginPlayback()`.
- **Notes:** User must ensure buffer is DMA-safe (conventional memory, below 1 MB).

### SOUNDSCAPE_ServiceInterrupt
- **Signature:** `static void __interrupt __far SOUNDSCAPE_ServiceInterrupt(void)`
- **Purpose:** ISR called by hardware when half-buffer is transferred. Swaps buffer pointer, invokes callback, sends EOI.
- **Inputs:** None (standard interrupt handler).
- **Outputs/Return:** None.
- **Side effects:** Updates `SOUNDSCAPE_CurrentDMABuffer` (wraps at end), calls user `SOUNDSCAPE_CallBack()`, sends EOI to PIC. Manages interrupt stack switching.
- **Calls:** `GetStack()`, `SetStack()`, user callback function, `outp()`.
- **Notes:** Memory-locked function (must not page). Saves/restores stack. Validates interrupt ownership before servicing. Chains to old interrupt if not ours.

### SOUNDSCAPE_FindCard
- **Signature:** `static int SOUNDSCAPE_FindCard(void)`
- **Purpose:** Detect hardware presence by parsing SNDSCAPE.INI and reading gate-array chip ID.
- **Inputs:** None (reads SNDSCAPE.INI from environment or default path).
- **Outputs/Return:** `SOUNDSCAPE_Ok` if card found and config loaded, error code otherwise.
- **Side effects:** Fills `SOUNDSCAPE_Config` struct (BasePort, WavePort, DMAChan, WaveIRQ, MIDIIRQ, ChipID, etc.). Sets `SOUNDSCAPE_FoundCard = TRUE` on success.
- **Calls:** `parse()`, `strtol()`, `strstr()`, `DMA_VerifyChannel()`, `outp()`, `inp()`, `ga_read()`.
- **Notes:** Caches result; subsequent calls return immediately. Determines chip type (ODIE/OPUS/MMIC) via gate-array register read.

### SOUNDSCAPE_Setup
- **Signature:** `static int SOUNDSCAPE_Setup(void)`
- **Purpose:** Configure hardware registers, interrupt handler, DMA polarity, and AD-1848 codec.
- **Inputs:** None.
- **Outputs/Return:** `SOUNDSCAPE_Ok` or error code.
- **Side effects:** Writes gate-array and AD-1848 registers, saves old interrupt vector, installs new handler, sets IRQ mask.
- **Calls:** `SOUNDSCAPE_DisableInterrupt()`, `SOUNDSCAPE_StopPlayback()`, `ad_read()`, `ad_write()`, `ga_write()`, `outp()`, `inp()`, `_dos_getvect()`, `_dos_setvect()`, `IRQ_SetVector()`, `SOUNDSCAPE_EnableInterrupt()`, `tdelay()`.
- **Notes:** Handles chip-specific signal routing; skipped for MMIC. Sets max volume, enables interrupt pin, clears pending interrupts.

### SOUNDSCAPE_SetPlaybackRate
- **Signature:** `void SOUNDSCAPE_SetPlaybackRate(unsigned rate)`
- **Purpose:** Set AD-1848 sample rate; normalizes input to supported values (11025/22050/44100 Hz).
- **Inputs:** `rate` in Hz (any value < 20000 → 11025, 20000–29999 → 22050, ≥ 30000 → 44100).
- **Outputs/Return:** None.
- **Side effects:** Updates `SOUNDSCAPE_SampleRate`, calls `pcm_format()` to write codec register.
- **Calls:** `pcm_format()`.
- **Notes:** Must be called before playback starts.

### SOUNDSCAPE_SetMixMode
- **Signature:** `int SOUNDSCAPE_SetMixMode(int mode)`
- **Purpose:** Set audio format (mono/stereo, 8/16-bit).
- **Inputs:** `mode` bitmask (MONO/STEREO, EIGHT_BIT/SIXTEEN_BIT).
- **Outputs/Return:** Validated mode value.
- **Side effects:** Updates `SOUNDSCAPE_MixMode` and `SOUNDSCAPE_SamplePacketSize`, calls `pcm_format()`.
- **Calls:** `pcm_format()`.
- **Notes:** Affects DMA transfer length calculations.

### SOUNDSCAPE_StopPlayback
- **Signature:** `void SOUNDSCAPE_StopPlayback(void)`
- **Purpose:** Halt DMA and AD-1848 playback immediately.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Disables interrupts, stops AD-1848 (clears config register), disables DMA channel, sets `SOUNDSCAPE_SoundPlaying = FALSE`, clears buffer pointers.
- **Calls:** `SOUNDSCAPE_DisableInterrupt()`, `ad_write()`, `tdelay()`, `DMA_EndTransfer()`.
- **Notes:** Safe to call multiple times or when not playing.

### SOUNDSCAPE_GetCurrentPos
- **Signature:** `int SOUNDSCAPE_GetCurrentPos(void)`
- **Purpose:** Query current playback position within the active DMA buffer.
- **Inputs:** None.
- **Outputs/Return:** Offset in samples from start of current buffer, or error code if not playing.
- **Side effects:** Reads DMA controller registers via `DMA_GetCurrentPos()`.
- **Calls:** `DMA_GetCurrentPos()`.
- **Notes:** Accounts for sample width and stereo; returns offset in samples (not bytes).

### pcm_format
- **Signature:** `static void pcm_format(void)`
- **Purpose:** Write AD-1848 PCM format register based on current sample rate and mix mode.
- **Inputs:** None (uses `SOUNDSCAPE_SampleRate` and `SOUNDSCAPE_MixMode`).
- **Outputs/Return:** None.
- **Side effects:** Enters AD-1848 mode-change state, writes format register, delays for re-synch and autocalibration.
- **Calls:** `outp()`, `tdelay()`.
- **Notes:** Must disable interrupts externally; enables mode-change flag (0x40) in register address.

### ga_read / ga_write
- **Signature:** `static int ga_read(int rnum)` / `static void ga_write(int rnum, int value)`
- **Purpose:** Read/write Ensoniq gate-array indirect registers.
- **Inputs:** `rnum` = register number, `value` = data to write.
- **Outputs/Return:** Data read (ga_read).
- **Side effects:** I/O port access via `outp()`/`inp()`.
- **Calls:** `outp()`, `inp()`.
- **Notes:** No synchronization; callers must ensure safe access.

### ad_read / ad_write
- **Signature:** `static int ad_read(int rnum)` / `static void ad_write(int rnum, int value)`
- **Purpose:** Read/write AD-1848 codec indirect registers.
- **Inputs:** `rnum` = register number, `value` = data to write.
- **Outputs/Return:** Data read (ad_read).
- **Side effects:** I/O port access via `outp()`/`inp()`.
- **Calls:** `outp()`, `inp()`.
- **Notes:** Comments warn not to use during mode-change enable; no enforcement.

### SOUNDSCAPE_LockMemory / SOUNDSCAPE_UnlockMemory
- **Signature:** `static int SOUNDSCAPE_LockMemory(void)` / `static void SOUNDSCAPE_UnlockMemory(void)`
- **Purpose:** Lock/unlock critical regions and variables in physical memory to prevent page faults during interrupt handling.
- **Inputs:** None.
- **Outputs/Return:** `SOUNDSCAPE_Ok` or `SOUNDSCAPE_DPMI_Error` (lock only).
- **Side effects:** DPMI function 0x600 (lock page) called on all critical data and ISR code range.
- **Calls:** `DPMI_LockMemoryRegion()`, `DPMI_Lock()`, `DPMI_UnlockMemoryRegion()`, `DPMI_Unlock()`.
- **Notes:** Locks region from `SOUNDSCAPE_LockStart` to `SOUNDSCAPE_LockEnd` (ISR functions) and all global state.

### allocateTimerStack / deallocateTimerStack
- **Signature:** `static unsigned short allocateTimerStack(unsigned short size)` / `static void deallocateTimerStack(unsigned short selector)`
- **Purpose:** Allocate/deallocate conventional (DOS) memory for interrupt handler dedicated stack via DPMI 0x100/0x101.
- **Inputs:** `size` in bytes for allocate, `selector` for deallocate.
- **Outputs/Return:** Selector (real-mode segment descriptor) or NULL on failure.
- **Side effects:** DPMI calls; memory is locked.
- **Calls:** `int386()` (DPMI calls).
- **Notes:** Stack must be in conventional memory to avoid nesting through protected/real mode transitions.

## Control Flow Notes

**Initialization phase (SOUNDSCAPE_Init → SOUNDSCAPE_Setup):**
- Detect card via config file
- Lock memory to prevent ISR page faults
- Install interrupt vector
- Configure gate-array and AD-1848 hardware
- Ready for playback

**Frame / Playback phase:**
- User calls `SOUNDSCAPE_BeginBufferedPlayback()`
- DMA controller copies half-buffer to sound card
- Hardware raises interrupt at half-buffer boundary
- `SOUNDSCAPE_ServiceInterrupt()` fires:
  - Rotates `SOUNDSCAPE_CurrentDMABuffer` pointer
  - Invokes user callback (for buffer refilling)
  - Sends EOI to PIC
- Loop continues until `SOUNDSCAPE_StopPlayback()` called

**Shutdown phase:**
- Halt DMA and AD-1848
- Restore original interrupt vector
- Unlock memory
- Deallocate stack

## External Dependencies
- **Includes:** `dos.h`, `conio.h`, `stdlib.h`, `stdio.h`, `string.h`, `ctype.h`, `time.h` (DOS runtime)
- **Local headers:** `interrup.h` (DisableInterrupts/RestoreInterrupts), `dpmi.h` (memory locking), `dma.h` (DMA controller), `irq.h` (IRQ setup), `sndscape.h` (public API)
- **External symbols (defined elsewhere):**
  - `DMA_SetupTransfer()`, `DMA_EndTransfer()`, `DMA_VerifyChannel()`, `DMA_GetCurrentPos()`, `DMA_ErrorString()` (dma.c)
  - `IRQ_SetVector()`, `IRQ_RestoreVector()` (irq.c)
  - `DPMI_LockMemoryRegion()`, `DPMI_Lock()`, etc. (dpmi.c)
  - `_dos_getvect()`, `_dos_setvect()`, `_chain_intr()`, `int386()` (DOS/compiler runtime)
  - `inp()`, `outp()` (compiler intrinsics for I/O ports)
