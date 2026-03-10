# audiolib/source/pas16.c

## File Purpose
Low-level hardware driver for Pro AudioSpectrum (PAS) sound cards on DOS/protected-mode systems. Manages interrupt handlers, DMA transfers, hardware state, and mixer control via a loadable real-mode driver (MVSOUND.SYS). Bridges protected-mode (386+) code to real-mode hardware and driver using DPMI calls.

## Core Responsibilities
- **Driver detection & initialization**: Verify MVSOUND.SYS driver presence, query DMA/IRQ configuration, auto-detect card port address.
- **Interrupt management**: Install/uninstall interrupt handlers for DMA completion, manage interrupt controller masks, chain to old handlers.
- **DMA setup & monitoring**: Configure DMA channels, manage circular DMA buffers, track playback position, coordinate with interrupt-driven transfers.
- **Hardware configuration**: Program sample rate timers, buffer counters, mix mode (mono/stereo/8/16-bit), audio filters, cross-channel controls.
- **Volume control**: PCM and FM mixer volume management via real-mode driver function calls.
- **State preservation**: Save/restore original hardware state and mixer settings across init/shutdown.
- **Protected-mode support**: Lock/unlock memory for real-mode access, allocate conventional stack for interrupt handler, use DPMI for real-mode calls.

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MVState` | struct (opaque, defined elsewhere) | Hardware state snapshot from MVSOUND.SYS (write-only register cache) |
| `MVFunc` | struct (opaque, defined elsewhere) | Function table with mixer and control function pointers from driver |
| `dpmi_regs` | struct (from dpmi.h) | x86 register state for DPMI real-mode function calls |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PAS_Interrupts` | `static const int[]` | static | Maps IRQ 0–15 to x86 interrupt vector numbers (e.g., IRQ 10 → 0x72) |
| `PAS_OldInt` | `static void (interrupt far *)()` | static | Saved original interrupt handler before installation |
| `PAS_IntController1Mask`, `PAS_IntController2Mask` | `static int` | static | Cached I/O port 0x21/0xA1 interrupt masks at init time |
| `PAS_Installed` | `static int` | static | Boolean: driver initialized |
| `PAS_TranslateCode` | `static int` | static | Port address offset for card I/O (auto-detected) |
| `PAS_OriginalPCMLeftVolume`, `PAS_OriginalPCMRightVolume` | `static int` | static | Saved PCM mixer volumes on init |
| `PAS_OriginalFMLeftVolume`, `PAS_OriginalFMRightVolume` | `static int` | static | Saved FM mixer volumes on init |
| `PAS_DMAChannel`, `PAS_Irq` | `static/extern int` | static | DMA channel and IRQ from card query |
| `PAS_State`, `PAS_Func` | `static MVState*`, `static MVFunc*` | static | Pointers to driver's state table and function table |
| `PAS_OriginalState` | `static MVState` | static | Copy of hardware state at init for restoration |
| `PAS_SampleSizeConfig` | `static int` | static | Cached sample size register value |
| `PAS_DMABuffer`, `PAS_DMABufferEnd`, `PAS_CurrentDMABuffer` | `static char*` | static | Circular DMA buffer pointers (managed by interrupt handler) |
| `PAS_TotalDMABufferSize` | `static int` | static | Total DMA buffer size in bytes |
| `PAS_TransferLength`, `PAS_MixMode`, `PAS_SampleRate`, `PAS_TimeInterval` | `static int` | static | Configuration: bytes per interrupt, mono/stereo/bitwidth, Hz, timer ticks |
| `PAS_SoundPlaying` | `volatile int` | static | Boolean: playback/record active; read by interrupt handler |
| `PAS_CallBack` | `void (*)(void)` | static | User callback invoked by interrupt at transfer completion |
| `StackSelector`, `StackPointer` | `static unsigned short/long` | static | Protected-mode selector and pointer for custom interrupt handler stack |
| `oldStackSelector`, `oldStackPointer` | `static unsigned short/long` | static | Saved original stack state before interrupt switch |
| `irqstatus` | `static int` | static | Interrupt status read within interrupt handler (avoids local variables with stack switch) |
| `PAS_ErrorCode` | global `int` | static | Last error code (accessible to caller) |

## Key Functions / Methods

### PAS_Init
- **Signature:** `int PAS_Init(void)`
- **Purpose:** Complete hardware and driver initialization. Called once at startup.
- **Inputs:** None.
- **Outputs/Return:** `PAS_Ok` on success; error code otherwise (e.g., `PAS_DriverNotFound`, `PAS_OutOfMemory`, `PAS_UnableToSetIrq`).
- **Side effects:** Saves interrupt masks, queries driver, allocates and locks conventional memory stack, installs interrupt handler, configures defaults (sample rate, mix mode), enables audio output. Sets `PAS_Installed = TRUE`.
- **Calls:** `PAS_CheckForDriver`, `PAS_GetStateTable`, `PAS_GetFunctionTable`, `PAS_GetCardSettings`, `PAS_FindCard`, `PAS_SaveState`, `PAS_LockMemory`, `allocateTimerStack`, `_dos_getvect`, `_dos_setvect`, `IRQ_SetVector`, `PAS_SetPlaybackRate`, `PAS_SetMixMode`, `PAS_Write`.
- **Notes:** Must run before any playback/record. Saves original PCM/FM volumes for restoration. On error, unlocks memory and deallocates stack to avoid leaks.

### PAS_Shutdown
- **Signature:** `void PAS_Shutdown(void)`
- **Purpose:** Cleanup: stop playback, restore interrupt handler and hardware state, unlock memory.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Halts DMA, restores original interrupt vector, clears playback state, restores PCM/FM mixer volumes, unlocks all locked memory regions, deallocates conventional stack. Sets `PAS_Installed = FALSE`.
- **Calls:** `PAS_StopPlayback`, `IRQ_RestoreVector`, `_dos_setvect`, `PAS_CallMVFunction`, `PAS_UnlockMemory`, `deallocateTimerStack`.
- **Notes:** Safe to call multiple times (checks `PAS_Installed`). Note: `PAS_RestoreState()` is commented out (DEBUG), so hardware registers are not restored.

### PAS_BeginBufferedPlayback
- **Signature:** `int PAS_BeginBufferedPlayback(char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void (*CallBackFunc)(void))`
- **Purpose:** Set up and start multi-buffered playback. User divides buffer into `NumDivisions` chunks; interrupt fires after each chunk transfers.
- **Inputs:** Audio buffer pointer, size in bytes, number of divisions, sample rate (Hz), mix mode (MONO_8BIT/STEREO_16BIT/etc.), callback function pointer.
- **Outputs/Return:** `PAS_Ok` on success; `PAS_DmaError` if DMA setup fails.
- **Side effects:** Stops any current playback, configures mix mode and sample rate, divides buffer, registers callback, sets up DMA for auto-init read mode, configures sample-rate timer and buffer counter, enables interrupt, marks `PAS_SoundPlaying = TRUE`.
- **Calls:** `PAS_StopPlayback`, `PAS_SetMixMode`, `PAS_SetPlaybackRate`, `PAS_SetCallBack`, `PAS_SetupDMABuffer`, `PAS_BeginTransfer`.
- **Notes:** Auto-init DMA means hardware repeats the same buffer indefinitely; interrupt handler advances `PAS_CurrentDMABuffer` to track position.

### PAS_BeginBufferedRecord
- **Signature:** `int PAS_BeginBufferedRecord(char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void (*CallBackFunc)(void))`
- **Purpose:** Identical to playback but for recording (ADC instead of DAC).
- **Inputs:** (Same as playback.)
- **Outputs/Return:** (Same as playback.)
- **Side effects:** Uses `DMA_AutoInitWrite` instead of `DMA_AutoInitRead`.
- **Calls:** (Same as playback, but `PAS_BeginTransfer( RECORD )`.)

### PAS_ServiceInterrupt
- **Signature:** `void interrupt far PAS_ServiceInterrupt(void)`
- **Purpose:** Interrupt service routine fired when each DMA transfer chunk completes. Updates playback position and invokes user callback.
- **Inputs:** (Called by CPU on interrupt.)
- **Outputs/Return:** None.
- **Side effects:** Switches stack to custom `StackSelector:StackPointer`, reads interrupt status, clears sample-buffer interrupt flag, advances `PAS_CurrentDMABuffer` (wraps on buffer end), calls user `PAS_CallBack`, sends End-of-Interrupt (EOI) to controller(s), restores original stack, chains to `PAS_OldInt` if interrupt not from PAS.
- **Calls:** `GetStack`, `SetStack`, `PAS_Read`, `PAS_Write`, `_chain_intr`.
- **Notes:** Uses custom stack to avoid overflowing user stack. `#ifdef USESTACK` gates the stack-switch logic. Must restore stack before chaining to avoid crashing old handler.

### PAS_SetupDMABuffer
- **Signature:** `int PAS_SetupDMABuffer(char *BufferPtr, int BufferSize, int mode)`
- **Purpose:** Configure DMA controller for audio transfer on the PAS card's DMA channel.
- **Inputs:** Buffer pointer, size (bytes), DMA mode (`DMA_AutoInitRead`, `DMA_AutoInitWrite`, etc.).
- **Outputs/Return:** `PAS_Ok` or `PAS_DmaError`.
- **Side effects:** Enables PAS DMA (sets register), calls `DMA_SetupTransfer`, caches buffer pointers and end address.
- **Calls:** `PAS_Write`, `DMA_SetupTransfer`.

### PAS_BeginTransfer
- **Signature:** `void PAS_BeginTransfer(int mode)` (`mode` = `PLAYBACK` or `RECORD`)
- **Purpose:** Start the DMA transfer and timers. Called after DMA buffer and mix settings are configured.
- **Inputs:** Mode (playback vs. record).
- **Outputs/Return:** None.
- **Side effects:** Programs sample-rate timer and buffer-count timer, enables interrupt, configures sample-size register (8/16-bit), cross-channel control (mono/stereo, ADC/DAC), filter setting (low-pass), enables audio mute flag, marks `PAS_SoundPlaying = TRUE`.
- **Calls:** `PAS_SetSampleRateTimer`, `PAS_SetSampleBufferCount`, `PAS_EnableInterrupt`, `PAS_Read`, `PAS_Write`, `PAS_GetFilterSetting`.

### PAS_StopPlayback
- **Signature:** `void PAS_StopPlayback(void)`
- **Purpose:** Halt audio transfer and clean up.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Disables interrupt, stops PCM DAC/ADC, disables 16-bit flag, ends DMA transfer, marks `PAS_SoundPlaying = FALSE`, clears `PAS_DMABuffer`.
- **Calls:** `PAS_DisableInterrupt`, `PAS_Write`, `PAS_Read`, `DMA_EndTransfer`.

### PAS_SetPlaybackRate
- **Signature:** `void PAS_SetPlaybackRate(unsigned rate)`
- **Purpose:** Set playback sample rate (Hz). Clamps to `PAS_MinSamplingRate` (4kHz) – `PAS_MaxSamplingRate` (44kHz).
- **Inputs:** Sample rate in Hz.
- **Outputs/Return:** None.
- **Side effects:** Computes timer interval, adjusts for stereo, caches actual rate (as `PAS_TimeInterval` and `PAS_SampleRate`).
- **Calls:** `CalcTimeInterval`, `CalcSamplingRate` (defined elsewhere).

### PAS_GetPlaybackRate
- **Signature:** `unsigned PAS_GetPlaybackRate(void)`
- **Purpose:** Query current sample rate.
- **Inputs:** None.
- **Outputs/Return:** Sample rate in Hz.

### PAS_SetMixMode
- **Signature:** `int PAS_SetMixMode(int mode)`
- **Purpose:** Configure playback mode (mono/stereo, 8/16-bit).
- **Inputs:** Mix mode bitmask (e.g., `MONO_8BIT`, `STEREO_16BIT`, masked by `PAS_MaxMixMode`).
- **Outputs/Return:** Effective mode (may be reduced if board revision 0 doesn't support 16-bit).
- **Side effects:** Checks board revision; forces mode to 8-bit if revision 0. Calls `PAS_SetPlaybackRate` to recalculate timer.

### PAS_SetMixMode
- **Signature:** `int PAS_SetMixMode(int mode)`
- **Purpose:** (see above)

### PAS_GetCurrentPos
- **Signature:** `int PAS_GetCurrentPos(void)`
- **Purpose:** Return playback position within the current DMA buffer chunk.
- **Inputs:** None.
- **Outputs/Return:** Offset in bytes (or error `PAS_Error`/`PAS_NoSoundPlaying`).
- **Side effects:** Queries DMA controller for current address, subtracts current buffer start, adjusts for sample size (shifts right if 16-bit and/or stereo).
- **Calls:** `DMA_GetCurrentPos`.

### PAS_SetSampleRateTimer, PAS_SetSampleBufferCount
- **Signature:** `void PAS_SetSampleRateTimer(void)`, `void PAS_SetSampleBufferCount(void)`
- **Purpose:** Program hardware timers. Sample-rate timer fires at playback frequency; buffer-count timer fires after a chunk transfers.
- **Inputs:** (Use global `PAS_TimeInterval`, `PAS_TransferLength`.)
- **Outputs/Return:** None.
- **Side effects:** Disable timer, select via `LocalTimerControl`, write LoByte/HiByte, enable timer in `AudioFilterControl`.
- **Calls:** `DisableInterrupts`, `RestoreInterrupts`, `PAS_Write`, `PAS_Read`.
- **Notes:** Must disable interrupts during write to avoid corruption. 16-bit DMA channels halve count.

### PAS_SetPCMVolume, PAS_GetPCMVolume
- **Signature:** `int PAS_SetPCMVolume(int volume)`, `int PAS_GetPCMVolume(void)`
- **Purpose:** Control/query PCM mixer volume (0–255 scale).
- **Inputs:** Volume (0–255, mapped to 0–100% via driver).
- **Outputs/Return:** Status or effective volume.
- **Side effects:** Calls real-mode driver functions for left and right channels.
- **Calls:** `PAS_CallMVFunction`.

### PAS_SetFMVolume, PAS_GetFMVolume
- **Signature:** `void PAS_SetFMVolume(int volume)`, `int PAS_GetFMVolume(void)`
- **Purpose:** Control/query FM synth mixer volume.
- **Inputs/Outputs:** (Analog to PCM.)

### PAS_GetCardInfo
- **Signature:** `int PAS_GetCardInfo(int *MaxSampleBits, int *MaxChannels)`
- **Purpose:** Query card capabilities (8/16-bit, mono/stereo support).
- **Inputs:** Pointers to output variables.
- **Outputs/Return:** `PAS_Ok` or error. Sets `*MaxSampleBits` and `*MaxChannels`.
- **Side effects:** Lazy-initializes driver if not yet checked. Checks board revision in `PAS_State->intrctlr`.

### PAS_CheckForDriver, PAS_GetStateTable, PAS_GetFunctionTable, PAS_GetCardSettings
- **Signature:** `int PAS_CheckForDriver(void)`, `MVState *PAS_GetStateTable(void)`, `MVFunc *PAS_GetFunctionTable(void)`, `int PAS_GetCardSettings(void)`
- **Purpose:** Query MVSOUND.SYS driver via interrupt 0x??. Verify presence, get state/function pointers, and get DMA/IRQ.
- **Inputs:** None.
- **Outputs/Return:** Status and/or pointers.
- **Side effects:** Call real-mode driver via `int386`/`int86` interrupt.
- **Calls:** `int386`, `int86` (hardware interrupt).

### PAS_FindCard
- **Signature:** `int PAS_FindCard(void)`
- **Purpose:** Auto-detect card port by testing known addresses.
- **Inputs:** None.
- **Outputs/Return:** `PAS_Ok` or `PAS_CardNotFound`. Sets `PAS_TranslateCode`.
- **Calls:** `PAS_TestAddress` (defined elsewhere).

### PAS_EnableInterrupt, PAS_DisableInterrupt
- **Signature:** `void PAS_EnableInterrupt(void)`, `void PAS_DisableInterrupt(void)`
- **Purpose:** Unmask/mask the PAS interrupt in the CPU's interrupt controller(s).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Reads/modifies I/O ports 0x21 (PIC1) and 0xA1 (PIC2), clears pending interrupts, writes to PAS `InterruptControl` register.
- **Calls:** `DisableInterrupts`, `RestoreInterrupts`, `inp`, `outp`, `PAS_Read`, `PAS_Write`.

### PAS_Write, PAS_Read
- **Signature:** `void PAS_Write(int Register, int Data)`, `int PAS_Read(int Register)`
- **Purpose:** Low-level I/O: write/read PAS register via translated port.
- **Inputs:** Register offset, data (write only).
- **Outputs/Return:** Data read (or nothing).
- **Side effects:** XORs register with `PAS_TranslateCode` for obfuscation, calls `outp`/`inp`.

### PAS_CallMVFunction
- **Signature:** `int PAS_CallMVFunction(unsigned long function, int ebx, int ecx, int edx)`
- **Purpose:** Call a real-mode driver function (e.g., mixer control) from protected mode using DPMI.
- **Inputs:** Function address (far pointer as ulong), register values (EBX, ECX, EDX).
- **Outputs/Return:** Result in EBX (& 0xff).
- **Side effects:** Constructs `dpmi_regs`, clears I/O segment registers, calls DPMI real-mode transition.
- **Calls:** `DPMI_CallRealModeFunction`.

### PAS_SaveState, PAS_RestoreState
- **Signature:** `void PAS_SaveState(void)`, `void PAS_RestoreState(void)`
- **Purpose:** Save/restore hardware register state (write-only registers are cached in `PAS_State`).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Backup/restore all control registers and timer values.
- **Calls:** `PAS_Write`, `PAS_Read`.

### PAS_LockMemory, PAS_UnlockMemory
- **Signature:** `int PAS_LockMemory(void)`, `void PAS_UnlockMemory(void)`
- **Purpose:** Lock/unlock memory for real-mode interrupt handler and data access. Required because real-mode ISR cannot page fault.
- **Inputs:** None.
- **Outputs/Return:** `DPMI_Ok` / error.
- **Side effects:** Locks code region (`PAS_LockStart` to `PAS_LockEnd`), all static variables, interrupt pointers.
- **Calls:** `DPMI_LockMemoryRegion`, `DPMI_Lock` macro (via individual calls).

### PAS_SaveMusicVolume, PAS_RestoreMusicVolume
- **Signature:** `int PAS_SaveMusicVolume(void)`, `void PAS_RestoreMusicVolume(void)`
- **Purpose:** Preserve and restore FM mixer volume across init/shutdown.
- **Inputs:** None.
- **Outputs/Return:** Status.
- **Side effects:** Queries/restores FM mixer on first call; caches values globally.
- **Calls:** `PAS_CallMVFunction`.

### allocateTimerStack, deallocateTimerStack
- **Signature:** `static unsigned short allocateTimerStack(unsigned short size)`, `static void deallocateTimerStack(unsigned short selector)`
- **Purpose:** Allocate/deallocate conventional (DOS) memory for interrupt handler stack via DPMI 0x100/0x101.
- **Inputs:** Size in bytes (or selector to deallocate).
- **Outputs/Return:** Selector (or nothing).
- **Side effects:** DPMI calls.

### Trivial Helper Functions
- `PAS_ErrorString`: Lookup error message string.
- `PAS_SetCallBack`: Cache user callback.
- `PAS_GetFilterSetting`: Return low-pass filter bits for sample rate.
- `GetStack`, `SetStack`: Inline asm to save/restore stack via `ss:esp` (pragma aux).
- `PAS_CallInt`: Inline asm to call int 2fh (pragma aux); unused in visible code.

## Control Flow Notes

**Initialization (PAS_Init):**
1. Query MVSOUND.SYS driver for state and function tables.
2. Query card DMA/IRQ configuration and auto-detect port address.
3. Save original hardware state and mixer volumes.
4. Lock all memory needed for interrupt handler (code + data).
5. Allocate conventional stack for ISR.
6. Install interrupt handler via `_dos_setvect` (or `IRQ_SetVector` for high IRQs).
7. Set default sample rate and mix mode.
8. Set `PAS_Installed = TRUE`.

**Playback Setup (PAS_BeginBufferedPlayback):**
1. Stop any current playback.
2. Apply mix-mode and sample-rate settings.
3. Set up DMA controller and cache buffer pointers.
4. Configure and enable sample-rate and buffer-count timers.
5. Enable interrupt and set cross-channel controls.
6. Mark `PAS_SoundPlaying = TRUE`.

**Interrupt Loop (during playback):**
1. `PAS_ServiceInterrupt` fires at end of each buffer chunk.
2. Advance `PAS_CurrentDMABuffer` to next chunk (wraps on buffer end).
3. Call user `PAS_CallBack` to signal chunk completion (app refills if needed).
4. Send EOI to PIC(s).
5. Restore stack and return or chain to old handler.

**Shutdown (PAS_Shutdown):**
1. Stop playback.
2. Restore original interrupt vector.
3. Restore original mixer volumes.
4. Unlock all memory.
5. Deallocate conventional stack.
6. Set `PAS_Installed = FALSE`.

## External Dependencies
- **C Standard Library:** `dos.h`, `conio.h` (x86 DOS), `stdlib.h`, `stdio.h`, `string.h`.
- **Custom DPMI Layer:** `dpmi.h` — DPMI 0x31 interrupt (real-mode calls, memory allocation).
- **Custom DMA Layer:** `dma.h` — DMA controller setup and state.
- **Interrupt Management:** `interrup.h`, `irq.h` — Interrupt flag control and high-IRQ vector setup.
- **Public Interface:** `pas16.h`, `_pas16.h` (private definitions).
- **Hardware Driver:** MVSOUND.SYS (real-mode DOS driver) — accessed via software interrupt (0x??) for mixer and state queries.
- **Symbols defined elsewhere:** `CalcTimeInterval`, `CalcSamplingRate`, `PAS_TestAddress` (used but not shown), `RECORD`, `PLAYBACK`, `MONO_8BIT`, `STEREO_16BIT`, `SampleBufferInterruptFlag`, `InterruptStatus`, `InterruptControl`, etc. (likely in `_pas16.h`).
