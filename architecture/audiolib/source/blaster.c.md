# audiolib/source/blaster.c

## File Purpose

Low-level Sound Blaster driver for DOS, supporting multiple card versions (1.xx, Pro, 2.xx, and 16). Handles DSP communication, DMA-based audio playback/recording, mixer control, and interrupt servicing for digitized sound output.

## Core Responsibilities

- DSP (Digital Signal Processor) communication via port I/O (read, write, reset, version detection)
- DMA buffer setup and management; tracks current playback position across circular buffer
- Hardware interrupt handling; chains to old handler if interrupt not from Sound Blaster
- Playback/record initiation tailored to DSP version (different command sequences for 1.xx, 2.xx, 4.xx)
- Audio format and sample-rate configuration (mono/stereo, 8-bit/16-bit, rate bounds per card)
- Mixer control: volume adjustment for voice/MIDI channels; stereo mode switching
- DPMI memory locking for interrupt-safe code and buffer regions
- Configuration parsing from BLASTER environment variable; card capability detection
- Error reporting and status tracking

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| BLASTER_CONFIG | struct | Holds card I/O address, IRQ, DMA channels (8/16-bit), MIDI port, and emulation address. |
| CARD_CAPABILITY | struct | Stores per-card capabilities: support flag, mixer presence, max mix mode, min/max sample rates. |
| dpmi_regs | struct | DPMI register state for real-mode function calls. |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| BLASTER_Config | BLASTER_CONFIG | global | Current card configuration (address, IRQ, DMA). |
| BLASTER_Card | CARD_CAPABILITY | global | Capabilities of detected card. |
| BLASTER_Installed | static int | static | Tracks whether driver is initialized. |
| BLASTER_Version | global int | global | DSP version (e.g., 0x0100, 0x0400). |
| BLASTER_DMABuffer | static char* | static | Base pointer to circular DMA buffer. |
| BLASTER_CurrentDMABuffer | static char* | static | Current write position in DMA buffer. |
| BLASTER_TotalDMABufferSize | static int | static | Total size of DMA buffer in bytes. |
| BLASTER_TransferLength | static int | static | Size of each DMA transfer block. |
| BLASTER_MixMode | static int | static | Current mix mode (mono/stereo, 8/16-bit). |
| BLASTER_SampleRate | static unsigned | static | Current playback/record sample rate (Hz). |
| BLASTER_SamplePacketSize | static int | static | Bytes per sample (1, 2, 2, or 4 depending on mix mode). |
| BLASTER_HaltTransferCommand | static unsigned | static | DSP command to stop transfer (varies by DSP version). |
| BLASTER_SoundPlaying | volatile int | global | Flag indicating active playback. |
| BLASTER_SoundRecording | volatile int | global | Flag indicating active recording. |
| BLASTER_CallBack | function pointer | global | User callback invoked per DMA interrupt. |
| BLASTER_OldInt | static function ptr | static | Saved original interrupt vector. |
| BLASTER_IntController1Mask | static int | static | Saved PIC1 interrupt mask (IRQs 0–7). |
| BLASTER_IntController2Mask | static int | static | Saved PIC2 interrupt mask (IRQs 8–15). |
| BLASTER_MixerType | static int | static | Card mixer type (0, SBPro, SB16). |
| BLASTER_OriginalVoiceVolumeLeft / Right | static int | static | Saved user voice volume settings. |
| BLASTER_OriginalMidiVolumeLeft / Right | static int | static | Saved user MIDI volume settings. |
| GlobalStatus | static int | static | DMA interrupt status register copy (used in interrupt handler). |
| StackSelector, StackPointer | static | static | Dedicated stack for interrupt handler; prevents stack overflow in protected mode. |

## Key Functions / Methods

### BLASTER_ServiceInterrupt
- **Signature:** `void __interrupt __far BLASTER_ServiceInterrupt(void)`
- **Purpose:** Handles hardware interrupt from Sound Blaster card at end of DMA block transfer.
- **Inputs:** None (called by hardware).
- **Outputs/Return:** None.
- **Side effects:** 
  - Swaps to dedicated interrupt stack (if USESTACK defined).
  - Advances BLASTER_CurrentDMABuffer; wraps to start if at end.
  - Invokes user callback via BLASTER_CallBack.
  - For DSP 1.xx, restarts transfer manually.
  - Sends EOI to PIC (Programmable Interrupt Controller).
- **Calls:** GetStack, SetStack, BLASTER_ReadMixer, BLASTER_DSP1xx_BeginPlayback, BLASTER_DSP1xx_BeginRecord, user callback.
- **Notes:** Critical section; reads DSP version and interrupt status to distinguish 8/16-bit DMA or non-Blaster interrupts. Chains to old handler if not a Sound Blaster interrupt.

### BLASTER_Init
- **Signature:** `int BLASTER_Init(void)`
- **Purpose:** Initialize the sound card: reset DSP, detect version, set up interrupt handler, lock memory for real-mode interrupt safety.
- **Inputs:** None.
- **Outputs/Return:** BLASTER_Ok or BLASTER_Error code.
- **Side effects:**
  - Saves interrupt masks; allocates and locks interrupt handler stack.
  - Calls BLASTER_ResetDSP, BLASTER_GetDSPVersion, BLASTER_LockMemory.
  - Installs BLASTER_ServiceInterrupt as interrupt handler.
  - Sets BLASTER_Installed = TRUE on success.
- **Calls:** BLASTER_ResetDSP, BLASTER_GetDSPVersion, BLASTER_SaveVoiceVolume, DMA_VerifyChannel, BLASTER_LockMemory, allocateTimerStack, _dos_getvect, _dos_setvect, IRQ_SetVector.
- **Notes:** Requires BLASTER_Config.Address to be set beforehand. Aborts if address undefined or interrupt/DMA verification fails.

### BLASTER_Shutdown
- **Signature:** `void BLASTER_Shutdown(void)`
- **Purpose:** Stop playback, restore system state, deallocate resources, and uninstall interrupt handler.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:**
  - Calls BLASTER_StopPlayback, BLASTER_RestoreVoiceVolume, BLASTER_ResetDSP.
  - Restores original interrupt vector; deallocates interrupt stack.
  - Unlocks memory; sets BLASTER_Installed = FALSE.
- **Calls:** BLASTER_StopPlayback, BLASTER_RestoreVoiceVolume, BLASTER_ResetDSP, BLASTER_UnlockMemory, deallocateTimerStack, IRQ_RestoreVector, _dos_setvect.
- **Notes:** Safe to call even if init failed or never called.

### BLASTER_BeginBufferedPlayback
- **Signature:** `int BLASTER_BeginBufferedPlayback(char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void (*CallBackFunc)(void))`
- **Purpose:** Start circular-buffered DMA playback; divides buffer into blocks triggering interrupts.
- **Inputs:** 
  - BufferStart: pointer to DMA-safe buffer.
  - BufferSize: total buffer size in bytes.
  - NumDivisions: number of equal blocks; interrupt fires per block.
  - SampleRate: playback rate in Hz.
  - MixMode: mono/stereo, 8/16-bit flags.
  - CallBackFunc: user function called on each DMA block completion.
- **Outputs/Return:** BLASTER_Ok or BLASTER_Error.
- **Side effects:**
  - Stops any existing playback; sets mix mode, sample rate, enables speaker.
  - Sets up DMA buffer; enables interrupts; programs DSP for playback.
- **Calls:** BLASTER_StopPlayback, BLASTER_SetMixMode, BLASTER_SetupDMABuffer, BLASTER_SetPlaybackRate, BLASTER_SetCallBack, BLASTER_EnableInterrupt, BLASTER_SpeakerOn, BLASTER_DSP1xx/2xx/4xx_BeginPlayback.
- **Notes:** Routes to version-specific playback function. TransferLength = BufferSize / NumDivisions.

### BLASTER_BeginBufferedRecord
- **Signature:** `int BLASTER_BeginBufferedRecord(char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void (*CallBackFunc)(void))`
- **Purpose:** Start circular-buffered DMA recording, symmetric to playback but disables speaker.
- **Inputs:** Same as BLASTER_BeginBufferedPlayback.
- **Outputs/Return:** BLASTER_Ok or BLASTER_Error.
- **Side effects:** Identical to playback except calls BLASTER_SpeakerOff and DMA_AutoInitWrite mode.
- **Calls:** BLASTER_StopPlayback, BLASTER_SetMixMode, BLASTER_SetupDMABuffer, BLASTER_SetPlaybackRate, BLASTER_SetCallBack, BLASTER_EnableInterrupt, BLASTER_SpeakerOff, BLASTER_DSP1xx/2xx/4xx_BeginRecord.
- **Notes:** Constraints depend on DSP version and card capabilities.

### BLASTER_StopPlayback
- **Signature:** `void BLASTER_StopPlayback(void)`
- **Purpose:** Halt DMA transfer and reset audio state.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:**
  - Disables interrupt; sends DSP halt command (or reset for old cards).
  - Ends DMA transfer via DMA_EndTransfer; turns off speaker.
  - Clears BLASTER_SoundPlaying and BLASTER_SoundRecording flags.
- **Calls:** BLASTER_DisableInterrupt, BLASTER_ResetDSP, BLASTER_WriteDSP, DMA_EndTransfer, BLASTER_SpeakerOff.
- **Notes:** Safe to call anytime; checks mix mode to select correct DMA channel.

### BLASTER_GetDSPVersion
- **Signature:** `int BLASTER_GetDSPVersion(void)`
- **Purpose:** Query DSP version and populate BLASTER_Card capabilities based on version.
- **Inputs:** None.
- **Outputs/Return:** Version number (e.g., 0x0400) or BLASTER_Error.
- **Side effects:**
  - Populates BLASTER_Card.IsSupported, HasMixer, MaxMixMode, MinSamplingRate, MaxSamplingRate.
  - Sets BLASTER_MixerType (0, SBPro, SB16).
- **Calls:** BLASTER_WriteDSP, BLASTER_ReadDSP.
- **Notes:** Branches on version ranges (1.xx, 2.xx, 3.xx, 4.xx+). Older cards may have mixer unavailable.

### BLASTER_SetPlaybackRate
- **Signature:** `void BLASTER_SetPlaybackRate(unsigned rate)`
- **Purpose:** Set sampling rate for playback/recording.
- **Inputs:** rate: desired sample rate in Hz.
- **Outputs/Return:** None.
- **Side effects:**
  - For DSP < 4.xx: computes time constant and writes DSP_SetTimeConstant command.
  - For DSP ≥ 4.xx: writes literal rate via DSP_Set_DA_Rate and DSP_Set_AD_Rate.
  - Clamps rate to card's min/max and updates BLASTER_SampleRate.
- **Calls:** BLASTER_WriteDSP.
- **Notes:** Actual rate may differ from requested; stored in BLASTER_SampleRate after conversion. Older cards use time-constant encoding.

### BLASTER_SetMixMode
- **Signature:** `int BLASTER_SetMixMode(int mode)`
- **Purpose:** Set audio format (mono/stereo, 8/16-bit) and configure mixer chip if needed.
- **Inputs:** mode: bitfield with STEREO and SIXTEEN_BIT flags.
- **Outputs/Return:** Effective mode (masked by card capabilities).
- **Side effects:**
  - Updates BLASTER_MixMode and BLASTER_SamplePacketSize.
  - For SBPro/SBPro2: writes mixer chip to enable/disable stereo via MIXER_SBProOutputSetting.
- **Calls:** BLASTER_WriteMixer, BLASTER_SetPlaybackRate.
- **Notes:** Clamps mode to card's MaxMixMode; recalculates packet size from mode.

### BLASTER_WriteDSP
- **Signature:** `int BLASTER_WriteDSP(unsigned data)`
- **Purpose:** Send a byte to the DSP command/data port with handshaking.
- **Inputs:** data: byte to write.
- **Outputs/Return:** BLASTER_Ok or BLASTER_Error.
- **Side effects:** Polls write port ready flag (0x80) with timeout; sets error code on failure.
- **Calls:** None (low-level I/O).
- **Notes:** Timeout is 0xFFFF; assumes card is responding. Returns error if port remains busy.

### BLASTER_ReadDSP
- **Signature:** `int BLASTER_ReadDSP(void)`
- **Purpose:** Read a byte from the DSP data-available port with handshaking.
- **Inputs:** None.
- **Outputs/Return:** Byte read or BLASTER_Error if timeout.
- **Side effects:** Polls data-available flag (0x80) with timeout; sets error code on failure.
- **Calls:** None (low-level I/O).
- **Notes:** Timeout is 0xFFFF.

### BLASTER_ResetDSP
- **Signature:** `int BLASTER_ResetDSP(void)`
- **Purpose:** Reset the DSP and wait for ready response.
- **Inputs:** None.
- **Outputs/Return:** BLASTER_Ok or BLASTER_CardNotReady.
- **Side effects:** Writes reset port (0x06), waits ~100 cycles, then polls for BLASTER_Ready (0xAA).
- **Calls:** BLASTER_ReadDSP.
- **Notes:** Essential initialization; commented-out code suggests earlier attempt to read port 0. Uses delay loop and retry counter.

### BLASTER_GetEnv
- **Signature:** `int BLASTER_GetEnv(BLASTER_CONFIG *Config)`
- **Purpose:** Parse BLASTER environment variable and extract configuration fields (A, I, D, H, T, P, E).
- **Inputs:** Pointer to BLASTER_CONFIG struct to fill.
- **Outputs/Return:** BLASTER_Ok or error (EnvNotFound, InvalidParameter).
- **Side effects:** Calls getenv("BLASTER"); parses hex/decimal fields via sscanf.
- **Calls:** getenv, toupper, isxdigit, sscanf.
- **Notes:** Format: "A220 I5 D1 H5 T6" (address, IRQ, DMA8, DMA16, type, MIDI, EMU). Skips unknown parameters.

### BLASTER_EnableInterrupt / BLASTER_DisableInterrupt
- **Signature:** `void BLASTER_EnableInterrupt(void)` / `void BLASTER_DisableInterrupt(void)`
- **Purpose:** Unmask/mask the card's IRQ at the Programmable Interrupt Controller (PIC).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:**
  - EnableInterrupt: clears bit in PIC1 (0x21) or PIC2 (0xA1) mask; cascades IRQ2 if IRQ > 7.
  - DisableInterrupt: restores saved masks (BLASTER_IntController1/2Mask).
- **Calls:** inp, outp.
- **Notes:** Requires saved interrupt masks from BLASTER_Init.

### BLASTER_SetupDMABuffer
- **Signature:** `int BLASTER_SetupDMABuffer(char *BufferPtr, int BufferSize, int mode)`
- **Purpose:** Configure DMA controller for audio transfer (playback or record).
- **Inputs:** BufferPtr: DMA-safe buffer; BufferSize: size; mode: DMA_AutoInitRead or DMA_AutoInitWrite.
- **Outputs/Return:** BLASTER_Ok or BLASTER_Error.
- **Side effects:** Calls DMA_SetupTransfer; stores pointers and size in static variables.
- **Calls:** DMA_SetupTransfer.
- **Notes:** Selects 8-bit or 16-bit DMA channel based on BLASTER_MixMode.

### BLASTER_GetCurrentPos
- **Signature:** `int BLASTER_GetCurrentPos(void)`
- **Purpose:** Return current offset within the DMA buffer during playback/record.
- **Inputs:** None.
- **Outputs/Return:** Offset in samples (0-indexed) or BLASTER_Error.
- **Side effects:** Calls DMA_GetCurrentPos; adjusts for 16-bit and stereo.
- **Calls:** DMA_GetCurrentPos.
- **Notes:** Offset is sample-count, adjusted by mix mode (divide by 2 for 16-bit, divide by 2 for stereo).

### Volume & Mixer Functions
- **BLASTER_SetVoiceVolume / BLASTER_GetVoiceVolume**
  - Set/get voice (digitized) sound channel volume (0–255).
  - Route to mixer depending on card type (SBPro, SB16).
  - BLASTER_SetVoiceVolume clamps to 0–255; packs left/right for SBPro.

- **BLASTER_SetMidiVolume / BLASTER_GetMidiVolume**
  - Set/get FM/MIDI channel volume.
  - Symmetric to voice functions.

- **BLASTER_SaveVoiceVolume / BLASTER_RestoreVoiceVolume**
  - Cache and restore user's voice volume on init/shutdown.

- **BLASTER_SaveMidiVolume / BLASTER_RestoreMidiVolume**
  - Cache and restore user's MIDI volume.

- **BLASTER_CardHasMixer**
  - Returns BLASTER_Card.HasMixer.

### DSP Version-Specific Playback Functions
- **BLASTER_DSP1xx_BeginPlayback / BLASTER_DSP1xx_BeginRecord**
  - Issue DSP_Old8BitDAC / DSP_Old8BitADC command; used for DSP 1.xx.
  - No auto-init; requires manual restart in interrupt handler.

- **BLASTER_DSP2xx_BeginPlayback / BLASTER_DSP2xx_BeginRecord**
  - Issue DSP_SetBlockLength then auto-init command (8-bit high-speed or normal).
  - Halts via DSP_Reset or DSP_Halt8bitTransfer depending on sample rate.

- **BLASTER_DSP4xx_BeginPlayback / BLASTER_DSP4xx_BeginRecord**
  - Support 16-bit and stereo modes.
  - Choose transfer command and mode flags based on BLASTER_MixMode.

### Configuration Functions
- **BLASTER_SetCardSettings / BLASTER_GetCardSettings**
  - Copy user config to/from BLASTER_Config; set BLASTER_MixerAddress and BLASTER_MixerType.
  - Default EMU address to card address + 0x400.

- **BLASTER_GetCardInfo**
  - Return max sample bits (8 or 16) and channels (1 or 2).

### Memory Locking
- **BLASTER_LockMemory / BLASTER_UnlockMemory**
  - Use DPMI calls to lock interrupt handler code and data in physical memory.
  - Prevents page faults during interrupt (which would crash in real-mode handler).
  - Locks code region from BLASTER_LockStart to BLASTER_LockEnd, plus all global/static variables.

### Helper Functions
- **allocateTimerStack / deallocateTimerStack**: Allocate conventional memory for interrupt stack via DPMI.
- **BLASTER_SetupWaveBlaster / BLASTER_ShutdownWaveBlaster**: Disable/restore MPU-401 interrupts on SB16 to avoid conflicts.
- **BLASTER_WriteMixer / BLASTER_ReadMixer**: Low-level mixer chip I/O (address/data port pair).
- **BLASTER_ErrorString**: Return human-readable error message.
- **BLASTER_SetCallBack**: Register user interrupt callback.
- **BLASTER_SpeakerOn / BLASTER_SpeakerOff**: Enable/disable DAC output via DSP command.

## Control Flow Notes

**Initialization:**
1. BLASTER_Init() is the entry point; must follow BLASTER_GetEnv() and BLASTER_SetCardSettings().
2. Resets DSP → queries version → installs interrupt handler → locks memory → ready for playback.

**Playback/Record:**
1. BLASTER_BeginBufferedPlayback/Record() sets up mix mode, sample rate, DMA buffer, and starts DSP transfer.
2. Hardware generates interrupt at each DMA block boundary.
3. BLASTER_ServiceInterrupt() fires in protected-mode context with dedicated stack; calls user callback.
4. Interrupt handler advances circular buffer pointer and, for DSP 1.xx, restarts transfer.

**Shutdown:**
1. BLASTER_Shutdown() stops playback, restores volumes, resets DSP, uninstalls interrupt handler, unlocks memory.
2. Safe to call anytime; defensive checks prevent double-shutdown.

## External Dependencies

- **DPMI** (dpmi.h): DPMI_LockMemory, DPMI_UnlockMemory, DPMI_UnlockMemoryRegion, DPMI_LockMemoryRegion for memory protection.
- **DMA** (dma.h): DMA_SetupTransfer, DMA_EndTransfer, DMA_GetCurrentPos, DMA_VerifyChannel, DMA_ErrorString for DMA controller setup.
- **IRQ** (irq.h): IRQ_SetVector, IRQ_RestoreVector for high-IRQ (8–15) interrupt installation.
- **DOS/Watcom built-ins**: inp, outp (port I/O), int386, _dos_getvect, _dos_setvect, _chain_intr (interrupt vectors).
- **Standard C**: getenv, sscanf, toupper, isxdigit, memset, min, max.
