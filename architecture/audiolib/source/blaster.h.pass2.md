# audiolib/source/blaster.h — Enhanced Analysis

## Architectural Role

BLASTER.H provides low-level hardware abstraction for Sound Blaster-compatible ISA audio cards (SB, SBPro, SB16 variants). It forms the PCM playback/recording layer of Vanilla ROTT's audio subsystem—situated below higher-level audio mixers and above raw hardware I/O and DMA controllers. The module enables buffered, interrupt-driven audio streaming with volume and format control, complementing the MIDI/FM synthesis subsystems (AL_MIDI, ADLIBFX) that handle game music and effects.

## Key Cross-References

### Incoming (who depends on this file)
- Game audio initialization and playback code indirectly via audio library wrappers
- Alternative implementation in `blastold.c` suggests this was a refactoring/replacement module
- Functions like BLASTER_Init, BLASTER_BeginBufferedPlayback are entry points for the audio subsystem

### Outgoing (what this file depends on)
- **DMA subsystem** (`dma.h/dma.c`): BLASTER_SetupDMABuffer configures DMA transfers; BLASTER_GetCurrentPos queries DMA progress
- **Hardware I/O** (implicit): ISA port I/O for DSP and mixer register access
- **Interrupt handling** (implicit): BLASTER_SetCallBack registers callback invoked at DMA completion; BLASTER_EnableInterrupt/DisableInterrupt manage IRQ state
- **Memory management** (implicit): BLASTER_LockMemory/UnlockMemory ensure DMA-safe buffers (DOS/DPMI era requirement)

## Design Patterns & Rationale

**Hardware Abstraction Layer (HAL)**: Encapsulates Sound Blaster register I/O (WriteDSP, ReadDSP, WriteMixer, ReadMixer) behind a hardware-agnostic API. This allows higher-level code to work with logical concepts (sample rate, mix mode, volume) rather than register addresses.

**Buffered, Callback-Driven I/O**: Implements a classic 1990s real-time audio pattern—circular DMA buffer divided into divisions, with hardware interrupt triggering a callback each time a division completes. Avoids busy-waiting and amortizes interrupt overhead.

**Configuration Structure** (BLASTER_CONFIG): Centralizes hardware parameters (I/O address, card type, IRQ, DMA channels) into a single struct, enabling environment variable parsing (BLASTER GetEnv) and runtime reconfiguration.

**Tradeoffs**: The callback mechanism is simpler than modern event queues but less flexible for complex audio processing chains; the configuration parsing from BLASTER environment variable is DOS-specific.

## Data Flow Through This File

1. **Initialization**: Application calls BLASTER_Init → parses hardware config (from BLASTER env var or manual SetCardSettings) → sets sample rate and mix mode → locks memory → registers interrupt handler.

2. **Playback**: Application fills buffer, calls BLASTER_BeginBufferedPlayback (with callback) → hardware DMAs buffer divisions to DSP → DSP streams to DAC → at end of each division, interrupt fires callback (e.g., to refill buffer).

3. **Control**: Application adjusts mixer via WriteMixer (e.g., volume, speaker on/off) or queries state via ReadMixer.

4. **Shutdown**: BLASTER_StopPlayback stops DMA, BLASTER_Shutdown tears down interrupts and restores hardware state.

## Learning Notes

**Idiomatic to 1990s ISA/DOS era**:
- Manual DMA buffer locking (DMA couldn't access paged/relocated memory)
- Environment variable for hardware auto-detection (BLASTER string standard)
- Interrupt-driven buffering predates modern async I/O APIs
- No abstraction over card variants (hardcoded register maps per SB type)

**Modern equivalents**:
- DirectSound (Win95+), ALSA (Linux), CoreAudio (macOS), PulseAudio wrap these details
- Circular buffers remain common, but OS-level APIs hide DMA setup
- Callbacks evolved into event loops and async APIs

**Engine-specific insights**:
- ROTT layers FM synthesis (ADLIBFX) and MIDI (AL_MIDI) on top; this module handles only PCM
- The blastold.c variant suggests iterative refinement—likely moved DSP timing or DMA handling

## Potential Issues

- **No visible bounds checking** on mix mode flags or sample rate values—assumes caller validates (CalcSamplingRate, CalcTimeConstant in _blaster.h likely handle this)
- **Configuration parsing fragility**: BLASTER_GetEnv parses environment variable with no obvious error recovery for malformed strings
- **Callback at interrupt level**: If callback performs blocking I/O or allocates memory, it could deadlock or corrupt heap
