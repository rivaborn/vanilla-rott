# audiolib/source/blastold.c — Enhanced Analysis

## Architectural Role

This file implements a **legacy Sound Blaster driver variant** running alongside or as an alternative to `blaster.c`. Rather than a single unified implementation, the audio subsystem offers multiple driver implementations for hardware compatibility and fallback—blastold.c targets older Sound Blaster revisions and may represent an earlier codebase revision kept for compatibility. It sits in the **hardware abstraction layer** between the game engine (via `audiolib/source/` public headers) and DOS/ISA hardware, providing the lowest-level DSP communication, DMA management, and interrupt handling needed for digital audio playback and recording on Sound Blaster cards.

## Key Cross-References

### Incoming (who depends on this file)
- **Game engine audio subsystem**: The game calls public functions via `blaster.h` (e.g., `BLASTER_Init`, `BLASTER_BeginBufferedPlayback`, `BLASTER_SetVoiceVolume`), which may route to either `blaster.c` or `blastold.c` depending on compilation or runtime selection.
- **Other audio drivers**: The broader audiolib supports multiple backends (ADLIBFX, AWE32, AL_MIDI) which may coexist or compete for the same resources (DMA channels, IRQ lines). blastold.c must coordinate with these to avoid conflicts.
- **Mixer subsystem**: Volume and mixer functions (e.g., `BLASTER_GetVoiceVolume`, `BLASTER_SetMidiVolume`) are called by the game audio pipeline to control PCM and MIDI levels dynamically.

### Outgoing (what this file depends on)
- **DMA subsystem** (`dma.h/dma.c`): Manages DMA controller setup, transfer state, and position queries. blastold.c calls `DMA_SetupTransfer`, `DMA_EndTransfer`, `DMA_GetCurrentPos`, `DMA_VerifyChannel` to orchestrate buffer transfers.
- **IRQ vector management** (`irq.h/irq.c`): Handles high-level interrupt registration via `IRQ_SetVector` and `IRQ_RestoreVector` (used for IRQ > 7).
- **Protected-mode memory locking** (`dpmi.h`): Uses DPMI calls to lock critical interrupt-handler code and DMA buffers in physical memory, essential for reliably delivering audio in protected mode without page faults stalling hardware.
- **DOS/platform layer**: Low-level port I/O (`inp`, `outp`), interrupt vectors (`_dos_getvect`, `_dos_setvect`), and register access for DSP reset, mixer control, and buffer acknowledgment.

## Design Patterns & Rationale

**Hardware Abstraction Layer (HAL) with Multiple Implementations:**
- Two variants (`blaster.c` and `blastold.c`) suggest iterative refinement or branching to support legacy hardware paths. The naming convention (`blastold.c`) indicates this is a preserved older version, likely maintained for compatibility with early SB revisions or as a fallback if the primary driver fails.

**Circular DMA Buffer with Interrupt-Driven Refill:**
- Audio is split into equal divisions (e.g., 2–4 segments). When DMA completes one segment, the ISR fires, increments `BLASTER_CurrentDMABuffer`, and invokes `BLASTER_CallBack` to signal the game to refill that segment. This is a classic real-time constraint pattern: the callback must complete quickly (under the next DMA interrupt latency) to avoid buffer underrun.

**Version-Specific Command Sequences (DSP 1.xx, 2.xx, 4.xx):**
- Rather than a polymorphic dispatch table, the code embeds version checks inline (e.g., `if (BLASTER_Version < DSP_Version2xx)`). This reflects the era's hardware diversity and suggests that dynamic polymorphism (vtables) was either unavailable or considered overkill for a small set of fixed card types.

**Stack Switching in Interrupt Handler:**
- The `#ifdef USESTACK` block switches to a pre-allocated DPMI stack during interrupt handling. This prevents stack exhaustion if the game's main stack is small or the interrupt handler is deep. It also ensures the stack doesn't cross page boundaries and trigger page faults mid-ISR—critical in protected mode.

**Configuration via Environment Variable:**
- The `BLASTER_GetEnv` function parses the DOS `BLASTER` environment variable (e.g., `A220 I5 D1`). This was the standard mechanism for hardware auto-configuration before Plug-and-Play; moving configuration outside the code made it user-friendly and hardware-agnostic.

## Data Flow Through This File

**Initialization phase:**
```
BLASTER environment variable
  ↓
BLASTER_GetEnv (parse address, IRQ, DMA8, DMA16, card type)
  ↓
BLASTER_Init (reset DSP, query version, save masks, lock memory, install ISR)
  ↓
Protected-mode stack allocated, interrupt vector chained
```

**Playback setup:**
```
Game engine
  ↓
BLASTER_BeginBufferedPlayback(buffer, size, numDivisions, sampleRate, mixMode, callback)
  ↓
BLASTER_SetMixMode (configure mono/stereo, 8/16-bit; write mixer if SBPro)
  ↓
BLASTER_SetPlaybackRate (DSP time-constant or direct rate command)
  ↓
BLASTER_SetupDMABuffer (set pointers, compute transfer length)
  ↓
BLASTER_EnableInterrupt (unmask IRQ at PIC)
  ↓
BLASTER_DSPxxx_BeginPlayback (DSP command sequence for version)
  ↓
DMA_SetupTransfer (start DMA channel)
```

**Per-DMA-completion (interrupt-driven):**
```
Hardware raises IRQ
  ↓
BLASTER_ServiceInterrupt (stack switch, acknowledge DSP/mixer)
  ↓
Advance BLASTER_CurrentDMABuffer (circular wrap)
  ↓
For DSP 1.xx: restart DSP playback command (auto-init not available)
  ↓
Invoke BLASTER_CallBack (game refills buffer segment)
  ↓
Send EOI, restore stack, return
```

**Shutdown:**
```
Game calls BLASTER_Shutdown
  ↓
BLASTER_StopPlayback (disable interrupts, DSP halt, DMA end)
  ↓
BLASTER_ResetDSP
  ↓
BLASTER_RestoreVoiceVolume (restore original mixer settings)
  ↓
Restore original ISR vector, restore IRQ masks
  ↓
BLASTER_UnlockMemory, deallocateTimerStack
```

## Learning Notes

**DOS-Era Hardware Pragmatism:**
- This driver exemplifies solutions to real constraints: ISA hardware with direct port I/O, no abstraction; interrupt handlers must run blindingly fast and cannot page fault, hence explicit memory locking. Modern engines use graphics APIs and audio frameworks; this one talks directly to hardware registers.

**Version-Driven Behavior (Not Inheritance):**
- Rather than OOP polymorphism, card capabilities are queried at init time and checked inline throughout (e.g., `if (BLASTER_Version >= DSP_Version4xx)`). This is idiomatic to low-level systems code where vtables and virtual calls are too heavyweight or discouraged.

**Circular Buffering as Real-Time Pattern:**
- The circular DMA buffer with callback is the **canonical real-time audio pattern** before ring buffers (used in modern ALSA, CoreAudio, etc.). It avoids malloc() during playback and guarantees deterministic latency if the callback completes within the interrupt period.

**Manual Interrupt Masking (PIC Manipulation):**
- `BLASTER_EnableInterrupt` and `BLASTER_DisableInterrupt` manually read/write the 8259 PIC mask registers (0x21, 0xA1) for IRQ0–7 and IRQ8–15. Modern OSes abstract this away; seeing bare `inp(0x21)` reveals the era's hardware-centric design.

**WaveBlaster Support as Modular Add-On:**
- The WaveBlaster MIDI daughterboard functions (`BLASTER_SetupWaveBlaster`, `BLASTER_ShutdownWaveBlaster`) show modular expansion: hardware could be extended without redesign, but required explicit driver awareness to avoid conflicts (e.g., disabling the MPU401 interrupt in the mixer).

**Comparison to Modern Engines:**
- Modern engines (Unity, Unreal) use audio engines with sample-rate-independent time-domain processing, format-agnostic codecs, and plugin systems. Rott's audiolib binds directly to ISA hardware, requires hard-coded sample rates, and mixes multiple drivers (BLASTER, ADLIB, GUS, AWE32) at the application level rather than via a unified plug-in interface.

## Potential Issues

**No Thread Safety:** 
- Global state like `BLASTER_SoundPlaying`, `BLASTER_CallBack`, and DMA buffer pointers are unprotected. If the game calls `BLASTER_BeginBufferedPlayback` or `BLASTER_Shutdown` from outside the main thread while the ISR fires, races and corruption are possible. However, single-threaded DOS/early Windows games wouldn't expose this.

**Timeout Loops with No Bounds on Hardware Failure:**
- `BLASTER_WriteDSP` and `BLASTER_ReadDSP` spin for 0xFFFF iterations waiting for DSP ready. If the card is truly unresponsive (removed, disabled BIOS), the loop consumes CPU for tens of milliseconds, freezing the game. A faster timeout or interrupt-driven handshake would improve responsiveness.

**Version Detection Heuristics:**
- `BLASTER_GetDSPVersion` sends the GET_VERSION command (0xE1) and reads two bytes; it assumes the response is well-formed and doesn't validate against known version ranges. Corrupted or emulated DSPs could return garbage, leading to undefined behavior (e.g., treating 0xFFFF as DSP 4.xx).

**Protected-Mode Stack Allocation Fragility:**
- The inline assembly for `GetStack`/`SetStack` directly manipulates SS and ESP. If the DPMI server or CPU context switches unpredictably, the saved stack selectors/pointers could become stale, causing ISR crashes. Most DOSen ran single-threaded, but under DPMI multitasking (rare), this is unsafe.

---
