# audiolib/source/blaster.c — Enhanced Analysis

## Architectural Role

This file is the **hardware abstraction layer (HAL) for Sound Blaster card families** in a 1990s DOS audio subsystem. It sits between a higher-level audio mixing/playback manager (likely in audiolib or game code) and three lower-level subsystems: DMA controller, IRQ handler registry, and DPMI real-mode protection. The driver enables the game engine to play digitized sound effects and voice-overs by managing card-specific initialization, DMA-based circular buffering, and interrupt-driven sample delivery across four distinct hardware generations (1.xx → Pro → 2.xx → 16-bit).

## Key Cross-References

### Incoming (who depends on this file)

- **audiolib upper layer** (inferred from header exports): calls `BLASTER_Init`, `BLASTER_Shutdown`, `BLASTER_BeginBufferedPlayback/Record`, `BLASTER_SetMixMode`, `BLASTER_SetPlaybackRate`, `BLASTER_SetVoiceVolume` to initialize the card and manage playback state.
- **Game/audio manager** (inferred): registers callback functions via `BLASTER_SetCallBack` to refill or process audio buffers per DMA interrupt.
- **blastold.c** (legacy driver): duplicates many functions, suggesting this file replaced an older implementation while maintaining binary compatibility.

### Outgoing (what this file depends on)

- **DMA subsystem** (`audiolib/source/dma.c`): `DMA_SetupTransfer`, `DMA_EndTransfer`, `DMA_GetCurrentPos`, `DMA_VerifyChannel`, `DMA_ErrorString` — manages 8-bit and 16-bit DMA channels, circular mode setup, and block-transfer acknowledgment.
- **IRQ subsystem** (`audiolib/source/irq.c`): `IRQ_SetVector`, `IRQ_RestoreVector` — installs/removes high-IRQ (8–15) handlers via DPMI.
- **DPMI layer** (`dpmi.h`): memory locking operations prevent real-mode interrupt handler code/data from being paged out.
- **DOS/Watcom libraries**: low-level I/O (`inp`, `outp`), environment vars (`getenv`), interrupt vectors (`_dos_getvect`, `_dos_setvect`, `_chain_intr`).

## Design Patterns & Rationale

**Version-Conditional Branching**
- The driver implements distinct DSP command sequences for 1.xx, 2.xx, and 4.xx cards. Why? Early Sound Blaster versions lacked auto-init DMA, requiring manual transfer restart per interrupt; SB2.xx added auto-init (still 8-bit); SB16 added 16-bit DMA and mixer control. This branching is data-driven via DSP version detection (lines ~0x0100, 0x0200, 0x0400).

**Circular DMA Buffer with Per-Block Callbacks**
- Instead of callback-per-sample (too frequent), divides the buffer into N equal blocks and fires interrupt only at block boundaries. Reduces interrupt overhead and gives audio clients predictable sync points. Reflects the "producer-consumer" pattern common in DOS real-time audio.

**Dedicated Interrupt Stack** (ifdef USESTACK)
- Rather than use the caller's stack during interrupt, swaps to a pre-allocated 2KB stack (lines ~134–150). This prevents stack overflow in protected mode and is idiomatic for 386+ DOS drivers. The swap is implemented via inline Watcom assembly (`GetStack`/`SetStack` pragmas).

**Configuration via Environment Variable**
- Parses `BLASTER=A220 I5 D1 H5 T6` to extract I/O base, IRQ, DMA channels, and card type. Reflects the era's lack of plug-and-play; users had to manually specify hardware settings. The parser is forgiving (skips unknown tokens).

**Saved Interrupt Masks & Restoration**
- Caches PIC interrupt masks (lines ~BLASTER_IntController1/2Mask) before modifying, then restores them on disable. Ensures the driver doesn't orphan other device IRQs.

## Data Flow Through This File

```
User App
    ↓
BLASTER_Init() → Reset DSP, detect version, install interrupt handler, lock memory
    ↓
BLASTER_BeginBufferedPlayback(buffer, size, divisions, rate, mixmode, callback)
    ├→ BLASTER_SetMixMode() → configure mixer chip (if SBPro/16)
    ├→ BLASTER_SetPlaybackRate() → set DSP time constant or rate register
    ├→ BLASTER_SetupDMABuffer() → configure DMA controller, select 8/16-bit channel
    ├→ BLASTER_EnableInterrupt() → unmask IRQ at PIC
    └→ BLASTER_DSPxxxx_BeginPlayback() → send DSP play command
         ↓
[Hardware DMA & Audio Output]
         ↓
    [End of DMA Block]
         ↓
BLASTER_ServiceInterrupt() [PROTECTED MODE, DEDICATED STACK]
    ├→ Acknowledge interrupt at mixer/DSP
    ├→ Advance BLASTER_CurrentDMABuffer pointer (wrap if needed)
    ├→ Call user BLASTER_CallBack() [buffer-refill opportunity]
    └→ For DSP 1.xx: manually restart transfer
         ↓
[Next DMA block begins...]
```

**Key state machine:** `BLASTER_SoundPlaying` and `BLASTER_SoundRecording` flags gate the ISR logic; `BLASTER_CurrentDMABuffer` tracks read/write position. On shutdown, `BLASTER_StopPlayback` → `BLASTER_ResetDSP` → `BLASTER_Shutdown` chain reverses initialization (restore volumes, uninstall handler, unlock memory).

## Learning Notes

**Idiomatic DOS Audio Driver Patterns**
- This driver exemplifies 1990s real-time audio on x86: direct port I/O, polled handshaking (0x80 busy flag), interrupt-driven DMA, and explicit memory locking to avoid page faults in real-mode handlers.
- The version detection strategy (querying DSP version 0xE1 command) is standard for Sound Blaster; hardcoding per-version behavior is more efficient than runtime capability discovery.
- Circular buffers with interrupt callbacks predate modern OS-level audio APIs (ALSA, CoreAudio, WASAPI); this pattern evolved into the "ring buffer" model still used in real-time audio.

**What Modern Engines Do Differently**
- Modern engines abstract audio via OS-level APIs (DirectSound, Core Audio, ALSA, PulseAudio) rather than direct hardware; no manual DMA setup or interrupt handling.
- Memory locking is implicit in managed memory; stack switching is unnecessary (OS scheduler handles stack per thread/context).
- Mixer control is OS/driver-managed, not application-managed.
- Version detection is automated (PnP/driver enumeration), not manual environment variables.

**Cross-Cutting Insight: Tight Coupling to Hardware Versions**
The first-pass analysis noted "version-specific branching," but the cross-reference reveals the broader implication: **this driver only abstracts Sound Blaster, not generic audio**. There is *no* device-driver abstraction layer; a different sound card would require a completely separate driver. Compare this to modern driver architecture (e.g., ALSA in Linux) where a common API masks card differences. The presence of `blastold.c` suggests Apogee maintained multiple drivers in parallel—a maintainability burden visible in the codebase structure itself.

## Potential Issues

1. **Stack Switch Complexity** (lines ~USESTACK): The inline assembly for stack swapping is fragile; if the compiler changes calling convention or the pragma is misunderstood, the interrupt handler could corrupt memory. No fallback if `GetStack`/`SetStack` fail.

2. **Incomplete Interrupt Detection (DSP < 4.xx)** (lines ~305–320): Older cards cannot distinguish 8-bit vs. 16-bit DMA or non-Blaster interrupts; the handler falls through to `_chain_intr`, but if chaining is messed up, interrupt storms are possible.

3. **Timeout Constants Hardcoded** (lines ~0xFFFF in WriteDSP/ReadDSP): No adaptive retry logic; if the card is slow or hung, the driver wedges for ~65K iterations, blocking the entire system. On a 486, this could stall the game visibly.

4. **No Resource Cleanup on Partial Init Failure**: If `BLASTER_Init` fails partway (e.g., interrupt installation succeeds but memory locking fails), some resources leak (allocated stack, installed vector not restored). The code relies on `BLASTER_Installed` flag, but doesn't guard re-entry.

5. **Global Callback State**: Only one `BLASTER_CallBack` per instance; no way to multiplex multiple audio streams at the DMA level. The upper layer must handle mixing in software.
