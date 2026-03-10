# audiolib/source/blastold.c

## File Purpose

Low-level driver for Sound Blaster sound cards (SB 1.0 through SB16) in protected mode (DOS). Manages DSP communication, DMA transfers, interrupt handling, and mixer control to enable digital audio playback and recording.

## Core Responsibilities

- Initialize and shut down Sound Blaster hardware via DSP command sequences
- Handle hardware interrupts on DMA completion, manage circular audio buffers, and invoke user callbacks
- Parse BLASTER environment variable and validate card configuration
- Set up and manage DMA channels for audio data transfer (8-bit and 16-bit)
- Support multiple card types with version-specific DSP command sets (1.xx, 2.xx, 4.xx)
- Control mixer chip for volume and audio source management
- Lock/unlock kernel memory and allocate protected-mode stack for interrupt handler
- Provide error reporting and environment variable handling

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `BLASTER_CONFIG` | struct | Hardware configuration (address, IRQ, DMA channels, MIDI port, card type) |
| `CARD_CAPABILITY` | struct | Per-card-type capabilities (supported mix modes, sample rate limits, mixer availability) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `BLASTER_Config` | `BLASTER_CONFIG` | global | Current card configuration |
| `BLASTER_Installed` | static int | static | Initialization flag |
| `BLASTER_Version` | static int | static | DSP version number (e.g., 0x0400 for SB16) |
| `BLASTER_DMABuffer` | static char* | static | Start of DMA buffer for audio data |
| `BLASTER_CurrentDMABuffer` | static char* | static | Current write/read position in circular buffer |
| `BLASTER_DMABufferEnd` | static char* | static | End boundary of DMA buffer |
| `BLASTER_TotalDMABufferSize` | static int | static | Total allocated buffer size |
| `BLASTER_TransferLength` | static int | static | Bytes per DMA transfer (buffer / NumDivisions) |
| `BLASTER_MixMode` | static int | static | Current mix mode (mono/stereo, 8/16-bit) |
| `BLASTER_SamplePacketSize` | static int | static | Bytes per sample (1, 2, or 4 depending on mode) |
| `BLASTER_SampleRate` | static unsigned | static | Current playback rate in Hz |
| `BLASTER_HaltTransferCommand` | static unsigned | static | DSP halt command (mode-specific) |
| `BLASTER_SoundPlaying` | volatile int | global | Set by init, cleared by interrupt or stop; user-readable |
| `BLASTER_SoundRecording` | volatile int | global | Set by init, cleared by interrupt or stop; user-readable |
| `BLASTER_CallBack` | function pointer | global | User callback invoked on each DMA completion |
| `BLASTER_OldInt` | static function pointer | static | Saved original interrupt vector for restoration |
| `BLASTER_IntController1Mask` | static int | static | Saved IRQ mask for PIC controller 1 (IRQ 0–7) |
| `BLASTER_IntController2Mask` | static int | static | Saved IRQ mask for PIC controller 2 (IRQ 8–15) |
| `BLASTER_MixerAddress` | static int | static | I/O port base for mixer chip |
| `BLASTER_MixerType` | static int | static | Card type for mixer register selection |
| `BLASTER_OriginalMidiVolumeLeft` / `*Right` | static int | static | Saved MIDI volume before init |
| `BLASTER_OriginalVoiceVolumeLeft` / `*Right` | static int | static | Saved voice volume before init |
| `BLASTER_WaveBlasterPort` | static int | static | MIDI port for WaveBlaster daughterboard |
| `BLASTER_WaveBlasterState` | static int | static | Saved mixer state for WaveBlaster MPU401 |
| `StackSelector` / `StackPointer` | static | static | Protected-mode stack for interrupt handler (allocated from DPMI) |
| `GlobalStatus` | static int | static | Temporary for interrupt handler (avoids stack use during stack switch) |
| `BLASTER_ErrorCode` | global int | global | Last error from this module |

**Const tables:**
- `BLASTER_Interrupts[]` – maps IRQ 0–15 to DOS interrupt vectors (or INVALID)
- `BLASTER_SampleSize[]` – sample packet sizes for each mix mode
- `BLASTER_CardConfig[]` – per-card-type capabilities

## Key Functions / Methods

### BLASTER_Init
- **Signature:** `int BLASTER_Init(void)`
- **Purpose:** Initialize Sound Blaster hardware: reset DSP, detect version, set up interrupt handler, lock memory, allocate protected-mode stack
- **Inputs:** None (uses global `BLASTER_Config`)
- **Outputs/Return:** `BLASTER_Ok` on success, error code otherwise
- **Side effects:** 
  - Saves interrupt controller masks
  - Allocates and locks memory (via DPMI)
  - Installs interrupt vector
  - Modifies protected-mode stack variables
  - Sets `BLASTER_Installed = TRUE`
- **Calls:** `BLASTER_ResetDSP`, `BLASTER_GetDSPVersion`, `BLASTER_SaveVoiceVolume`, `DMA_VerifyChannel`, `BLASTER_LockMemory`, `allocateTimerStack`, `_dos_getvect`, `_dos_setvect`, `IRQ_SetVector`
- **Notes:** If already installed, calls `BLASTER_Shutdown` first. Validates IRQ against `VALID_IRQ` macro.

### BLASTER_Shutdown
- **Signature:** `void BLASTER_Shutdown(void)`
- **Purpose:** Stop playback/recording, reset DSP, restore original interrupt and volume settings, unlock memory, deallocate stack
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** 
  - Calls `BLASTER_StopPlayback`, `BLASTER_RestoreVoiceVolume`, `BLASTER_ResetDSP`
  - Restores original interrupt vector
  - Unlocks all locked memory
  - Deallocates protected-mode stack
  - Sets `BLASTER_Installed = FALSE`
- **Calls:** `BLASTER_StopPlayback`, `BLASTER_RestoreVoiceVolume`, `BLASTER_ResetDSP`, `_dos_setvect`, `IRQ_RestoreVector`, `BLASTER_UnlockMemory`, `deallocateTimerStack`
- **Notes:** Safe to call multiple times; checks `BLASTER_Installed`.

### BLASTER_BeginBufferedPlayback
- **Signature:** `int BLASTER_BeginBufferedPlayback(char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void (*CallBackFunc)(void))`
- **Purpose:** Set up circular audio buffer and start DMA playback
- **Inputs:** 
  - `BufferStart`: audio data buffer
  - `BufferSize`: total buffer size
  - `NumDivisions`: number of equal divisions (determines DMA transfer length)
  - `SampleRate`: playback rate in Hz
  - `MixMode`: mono/stereo and 8/16-bit flags
  - `CallBackFunc`: callback on each DMA completion
- **Outputs/Return:** `BLASTER_Ok` on success, error code otherwise
- **Side effects:** 
  - Stops any running playback/recording
  - Sets mix mode, DMA buffer pointers, sample rate, callback
  - Enables interrupts, turns on speaker
  - Sets `BLASTER_SoundPlaying = TRUE`
- **Calls:** `BLASTER_StopPlayback`, `BLASTER_SetMixMode`, `BLASTER_SetupDMABuffer`, `BLASTER_SetPlaybackRate`, `BLASTER_SetCallBack`, `BLASTER_EnableInterrupt`, `BLASTER_SpeakerOn`, then version-specific playback start (`BLASTER_DSP1xx/2xx/4xx_BeginPlayback`)
- **Notes:** Transfer length is `BufferSize / NumDivisions`. Divides circular buffer into equal chunks for interrupt-driven updates. Commented-out code suggests prior conditional logic was removed.

### BLASTER_ServiceInterrupt
- **Signature:** `void __interrupt __far BLASTER_ServiceInterrupt(void)`
- **Purpose:** High-priority interrupt handler called on DMA completion; acknowledge hardware, advance circular buffer pointer, invoke user callback
- **Inputs:** None (processor state on interrupt)
- **Outputs/Return:** None; sends EOI to interrupt controller
- **Side effects:** 
  - Switches to protected-mode stack (if `USESTACK` defined)
  - Acknowledges interrupt at mixer/DSP (version-dependent)
  - Advances `BLASTER_CurrentDMABuffer` by `BLASTER_TransferLength`, wrapping at end
  - For older cards without auto-init DMA, restarts DSP playback/record
  - Invokes `BLASTER_CallBack()` if set
  - Sends EOI to PIC
- **Calls:** `GetStack`, `SetStack`, `BLASTER_ReadDSP`, `BLASTER_DSP1xx_BeginPlayback`, `BLASTER_DSP1xx_BeginRecord`, callback function (indirect), `_chain_intr`
- **Notes:** Critical code; must run fast. Uses inline assembly to manage stack. For SB16+, checks mixer ISR register to determine 8/16-bit source. May chain to old interrupt if not handled.

### BLASTER_BeginBufferedRecord
- **Signature:** `int BLASTER_BeginBufferedRecord(char *BufferStart, int BufferSize, int NumDivisions, unsigned SampleRate, int MixMode, void (*CallBackFunc)(void))`
- **Purpose:** Symmetric to `BLASTER_BeginBufferedPlayback` but for recording
- **Inputs:** Same as playback
- **Outputs/Return:** `BLASTER_Ok` on success, error code otherwise
- **Side effects:** Similar to playback; turns off speaker instead of on; sets `BLASTER_SoundRecording = TRUE`; calls record-specific DSP setup
- **Calls:** `BLASTER_StopPlayback`, `BLASTER_SetMixMode`, `BLASTER_SetupDMABuffer`, `BLASTER_SetPlaybackRate`, `BLASTER_SetCallBack`, `BLASTER_EnableInterrupt`, `BLASTER_SpeakerOff`, then version-specific record start
- **Notes:** Reuses same DMA infrastructure as playback but reverses buffer direction.

### BLASTER_WriteDSP / BLASTER_ReadDSP
- **Signature:** 
  - `int BLASTER_WriteDSP(unsigned data)`
  - `int BLASTER_ReadDSP(void)`
- **Purpose:** Send/receive single bytes to DSP with handshaking and timeout
- **Inputs/Outputs:** `data` (write) or return byte (read); `BLASTER_Error` if timeout
- **Side effects:** Busy-waits on I/O port, may set error code
- **Calls:** `inp`, `outp`, `BLASTER_SetErrorCode`
- **Notes:** Timeout count is 0xFFFF; old card detection may not work reliably.

### BLASTER_SetPlaybackRate
- **Signature:** `void BLASTER_SetPlaybackRate(unsigned rate)`
- **Purpose:** Configure DSP sample rate; older cards use time-constant formula, SB16 uses direct rate
- **Inputs:** `rate` in Hz
- **Outputs/Return:** None; updates `BLASTER_SampleRate`
- **Side effects:** 
  - Clamps rate to card capability range
  - For DSP < 4.xx: calculates time constant, updates stored rate via calculation
  - For DSP 4.xx: sets both playback and record rates via separate commands
- **Calls:** `BLASTER_WriteDSP` (multiple), `CalcTimeConstant`, `CalcSamplingRate` (macros)
- **Notes:** Rate adjustment is inexact for older cards due to time-constant quantization.

### BLASTER_SetMixMode
- **Signature:** `int BLASTER_SetMixMode(int mode)`
- **Purpose:** Set mono/stereo and 8/16-bit configuration; for SBPro, writes mixer chip
- **Inputs:** `mode` with `STEREO` and/or `SIXTEEN_BIT` flags
- **Outputs/Return:** Effective mode (masked by card capability)
- **Side effects:** 
  - Updates `BLASTER_MixMode` and `BLASTER_SamplePacketSize`
  - For SBPro/SBPro2: reads mixer, updates stereo bit, writes back
  - Resets playback rate for SBPro
- **Calls:** `BLASTER_WriteMixer`, `inp`, `outp`, `BLASTER_SetPlaybackRate`
- **Notes:** Bit depth may not be supported on older cards; automatically disabled by masking.

### BLASTER_StopPlayback
- **Signature:** `void BLASTER_StopPlayback(void)`
- **Purpose:** Halt DMA transfer and reset audio state
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** 
  - Disables interrupts
  - Sends DSP halt or reset command
  - Ends DMA transfer
  - Turns off speaker
  - Clears `BLASTER_SoundPlaying`, `BLASTER_SoundRecording`, and buffer pointer
- **Calls:** `BLASTER_DisableInterrupt`, `BLASTER_WriteDSP`, `BLASTER_ResetDSP`, `DMA_EndTransfer`, `BLASTER_SpeakerOff`
- **Notes:** Unconditional; safe to call when not playing.

### BLASTER_GetCurrentPos
- **Signature:** `int BLASTER_GetCurrentPos(void)`
- **Purpose:** Return playback position within current circular buffer segment
- **Inputs:** None
- **Outputs/Return:** Offset in samples from `BLASTER_CurrentDMABuffer` start; `BLASTER_Error` if not playing
- **Side effects:** Queries DMA controller for current address
- **Calls:** `DMA_GetCurrentPos`
- **Notes:** Adjusts for 16-bit and stereo sample sizes via right-shift.

### BLASTER_ResetDSP
- **Signature:** `int BLASTER_ResetDSP(void)`
- **Purpose:** Reset DSP to known state and verify acknowledgment
- **Inputs:** None
- **Outputs/Return:** `BLASTER_Ok` if reset acknowledged, `BLASTER_CardNotReady` otherwise
- **Side effects:** Outputs 1 then 0 to reset port, busy-waits, calls `BLASTER_ReadDSP`
- **Calls:** `BLASTER_ReadDSP`, `outp`
- **Notes:** Commented-out code suggests earlier complexity; final version does basic delay and reads READY (0xAA) response.

### BLASTER_GetDSPVersion
- **Signature:** `int BLASTER_GetDSPVersion(void)`
- **Purpose:** Query DSP version; used to detect card type at runtime
- **Inputs:** None
- **Outputs/Return:** Version as `(MajorVersion << 8) | MinorVersion` (e.g., 0x0400 for SB16), or `BLASTER_Error`
- **Side effects:** Sets error code if read fails
- **Calls:** `BLASTER_WriteDSP`, `BLASTER_ReadDSP`, `BLASTER_SetErrorCode`
- **Notes:** Version detection is critical for selecting command sequences.

### Mixer control functions
- **`BLASTER_WriteMixer(int reg, int data)` / `BLASTER_ReadMixer(int reg)`**
  - Write/read mixer registers for volume and feature control
  - Used by volume getter/setter for per-card adjustment (SBPro registers vs. SB16 left/right registers)
- **`BLASTER_GetVoiceVolume() / BLASTER_SetVoiceVolume(int volume)`**
  - Get/set voice/PCM channel volume (0–255)
  - Convert between card-specific register formats
- **`BLASTER_GetMidiVolume() / BLASTER_SetMidiVolume(int volume)`**
  - Get/set MIDI/FM synthesis volume
  - Same format conversion logic as voice
- **`BLASTER_SaveVoiceVolume() / BLASTER_RestoreVoiceVolume()`**
  - Save/restore user's voice volume at init/shutdown
- **`BLASTER_SaveMidiVolume() / BLASTER_RestoreMidiVolume()`**
  - Save/restore user's MIDI volume at init/shutdown
- **`BLASTER_CardHasMixer()`**
  - Check if current card type supports mixer; use cached or query via `BLASTER_GetEnv`

### Configuration and card detection
- **`BLASTER_GetEnv(BLASTER_CONFIG *Config)`**
  - Parse BLASTER environment variable (e.g., "A220 I5 D1 T6 H5 P330")
  - Validate card type, required parameters (address, IRQ, DMA8)
  - Return error if missing or invalid
- **`BLASTER_SetCardSettings(BLASTER_CONFIG Config)` / `BLASTER_GetCardSettings(BLASTER_CONFIG *Config)`**
  - Copy configuration to/from global `BLASTER_Config`
  - Shutdown if already installed before setting new config
- **`BLASTER_GetCardInfo(int *MaxSampleBits, int *MaxChannels)`**
  - Query card capability: max bits (8 or 16) and channels (1 or 2)
  - Used by caller to determine supported mix modes

### WaveBlaster MIDI support
- **`BLASTER_SetupWaveBlaster(int address)` / `BLASTER_ShutdownWaveBlaster()`**
  - Configure optional WaveBlaster daughterboard MIDI port
  - Disable/restore MPU401 interrupts in mixer to prevent conflicts with SB16 playback

### Error and callback
- **`BLASTER_ErrorString(int ErrorNumber)`**
  - Return human-readable error string for debugging
- **`BLASTER_SetCallBack(void (*func)(void))`**
  - Register user function called at end of each DMA transfer (for buffer refill logic)

### Memory and resource locking (protected mode support)
- **`BLASTER_LockMemory() / BLASTER_UnlockMemory()`**
  - Lock/unlock critical code and data to prevent page faults during interrupt handling
  - Uses DPMI calls to pin both function region (`BLASTER_LockStart` to `BLASTER_LockEnd`) and global variables
- **`allocateTimerStack() / deallocateTimerStack()`**
  - Use DPMI INT 31h to allocate/deallocate low-memory conventional memory for interrupt stack
  - Prevents stack overflow in interrupt handler

### DSP version-specific playback/record helpers
- **`BLASTER_DSP1xx_BeginPlayback(int length)` / `BLASTER_DSP1xx_BeginRecord(int length)`**
  - Start playback/record on DSP 1.xx without auto-init; requires re-trigger on each interrupt
- **`BLASTER_DSP2xx_BeginPlayback(int length)` / `BLASTER_DSP2xx_BeginRecord(int length)`**
  - Start playback/record on DSP 2.xx with auto-init mode
  - Switch to high-speed mode if sample rate and sample packet size exceed DSP_MaxNormalRate
- **`BLASTER_DSP4xx_BeginPlayback(int length)` / `BLASTER_DSP4xx_BeginRecord(int length)`**
  - Start playback/record on SB16 (DSP 4.xx) with 16-bit support
  - Select transfer command (8-bit or 16-bit) and signed/unsigned data format based on mix mode

### Interrupt controller helpers (macros and simple functions)
- **`BLASTER_EnableInterrupt() / BLASTER_DisableInterrupt()`**
  - Unmask/restore system IRQ at PIC; separate handling for IRQ < 8 (PIC 1) and IRQ ≥ 8 (PIC 2)
  - Uses saved masks from init time

## Control Flow Notes

**Initialization phase (`BLASTER_Init`):**
- Save interrupt masks → reset DSP → query version → configure default rate/mode → lock memory → allocate stack → install interrupt vector

**Playback/record setup (`BLASTER_BeginBufferedPlayback/Record`):**
- Set mix mode and sample rate → configure DMA buffer → enable interrupts → start DSP playback/record command sequence

**Per-buffer-completion (`BLASTER_ServiceInterrupt` on IRQ):**
- Switch stack → acknowledge DSP/mixer interrupt → advance circular buffer pointer (wrap at end) → trigger user callback → restore stack → send EOI to PIC

**Shutdown phase (`BLASTER_Shutdown`):**
- Stop playback → restore original volumes → reset DSP → restore interrupt vector and masks → unlock memory → deallocate stack

**Error handling:**
- All public functions check inputs and card state, set `BLASTER_ErrorCode`, return error code; callers check return and call `BLASTER_ErrorString` for user messages

## External Dependencies

- **dos.h, conio.h:** Low-level port I/O (`inp`, `outp`), DOS interrupt vectors (`_dos_getvect`, `_dos_setvect`), register structures (`union REGS`, `int386`)
- **stdlib.h, stdio.h, string.h, ctype.h:** Standard C utilities
- **dpmi.h:** Protected-mode memory locking and DPMI calls (defined elsewhere; uses `DPMI_Lock`, `DPMI_Unlock`, `DPMI_LockMemoryRegion`, `DPMI_UnlockMemoryRegion`)
- **dma.h:** DMA controller setup (defined elsewhere; uses `DMA_SetupTransfer`, `DMA_EndTransfer`, `DMA_GetCurrentPos`, `DMA_VerifyChannel`)
- **irq.h:** Higher IRQ vector support (defined elsewhere; uses `IRQ_SetVector`, `IRQ_RestoreVector`)
- **blaster.h:** Public interface and `BLASTER_CONFIG` struct
- **_blaster.h:** Private constants and `CARD_CAPABILITY` struct
- **Inline assembly pragmas:** `GetStack`, `SetStack` for stack manipulation in interrupt handler

**Not inferable from this file:** Actual implementation of DMA, DPMI, IRQ, and timer interrupt handling; depends on platform-specific modules.
