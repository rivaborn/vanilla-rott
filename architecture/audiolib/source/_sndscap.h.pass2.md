# audiolib/source/_sndscap.h — Enhanced Analysis

## Architectural Role

This private header defines the low-level hardware interface for **Ensoniq Soundscape** sound cards, one of several pluggable audio driver backends in the audio library. Unlike Blaster or AdLib devices (which appear heavily in cross-references), Soundscape represents an alternative PCM + MIDI card with dual-chip architecture (Ensoniq gate-array + AD-1848 codec). The file encapsulates all hardware-level register manipulation, DMA control, and interrupt handling to keep driver implementation details isolated from higher-level audio management layers.

## Key Cross-References

### Incoming (who depends on this file)
- **Isolated to `sndscape.c`** — The cross-reference index shows no functions from `_sndscap.h` exported or called elsewhere in the codebase. This is expected for a private header (`_` prefix), but suggests the **Soundscape driver is an alternative/supplementary backend**, not the primary audio path.
- Contrast with **BLASTER_*** and **AWE32_*** functions which appear throughout the cross-reference, indicating those drivers are primary targets.
- The public interface would be exposed via a parallel `sndscape.h` header (not shown here).

### Outgoing (what this file depends on)
- **x86 hardware I/O ports**: Interrupt controllers (0x20, 0xa0), DMA controller, Soundscape base I/O address (not defined here—passed at runtime)
- **DOS/DPMI memory services**: Memory locking for DMA (via `SOUNDSCAPE_LockMemory`), stack management for ISR context
- **Standard C**: `FILE` type for configuration parsing (in `parse()`)
- **No calls to other audio library subsystems** — hardware access is direct, not layered

## Design Patterns & Rationale

**Hardware Abstraction Layer (HAL)**  
Gate-array and codec register access is wrapped in `ga_read/ga_write` and `ad_read/ad_write` functions. This indirect addressing pattern (common in 1990s chips) is hidden behind a thin API, allowing register definitions to remain in the header while implementation details stay in the `.c` file.

**Pluggable driver model**  
The library supports multiple mutually-exclusive backends (Blaster, AWE32, Soundscape, etc.). Each has its own init/shutdown/service cycle. Soundscape's absence from the cross-reference suggests it's a **compile-time or runtime-selectable alternative** rather than a primary driver—users likely selected Blaster or AWE32 in production builds.

**Interrupt-driven DMA architecture**  
Follows 1990s ISA-era design: DMA buffer is set up, hardware is started, and `SOUNDSCAPE_ServiceInterrupt` responds to completion signals. This avoids CPU-intensive polling and is essential for real-time PCM playback on 386/486 hardware.

**DOS memory management**  
`SOUNDSCAPE_LockMemory/UnlockMemory` prevents DMA buffer pages from being swapped—critical in protected mode where the OS can move memory around. The stack management functions (`allocateTimerStack`/`deallocateTimerStack`) handle re-entrancy concerns for the ISR.

## Data Flow Through This File

1. **Initialization phase** (in `sndscape.c`):
   - `SOUNDSCAPE_FindCard()` → detects hardware at base address
   - `SOUNDSCAPE_Setup()` → configures gate-array and AD-1848 codec via `ga_write`/`ad_write`
   - `SOUNDSCAPE_LockMemory()` → locks DMA buffer in physical RAM

2. **Playback loop**:
   - Application calls `SOUNDSCAPE_SetupDMABuffer(buffer, size, mode)` → configures DMA and audio format
   - `SOUNDSCAPE_BeginPlayback(length)` → starts hardware playback
   - Hardware streams PCM data from buffer into DAC
   - DMA completion → CPU interrupt → `SOUNDSCAPE_ServiceInterrupt()` → ISR signals completion to application

3. **Shutdown**:
   - Stop playback, unlock memory, disable interrupts

Audio format is controlled via mode flags (MONO_8BIT, STEREO_16BIT, etc.) which parameterize register writes in `pcm_format()`.

## Learning Notes

**1990s PC audio architecture**  
Soundscape exemplifies high-end audio cards of that era: dual-chip design (gate-array for control + AD-1848 codec for PCM), DMA-based streaming, and interrupt-driven synchronization. Modern engines use continuous buffer ring formats and timer-based polling instead.

**DOS/protected mode constraints**  
The code shows explicit memory locking and far-pointer usage (`__far`), reflecting real-mode ↔ protected-mode transition challenges in 16-bit DOS extenders. Modern operating systems handle this transparently.

**Hardware indirection patterns**  
The gate-array's indirect addressing scheme (write register number to `GA_REGADDR`, then read/write data on `GA_REGDATA`) is repeated for the AD-1848 codec. This pattern conserves I/O port address space on ISA bus cards.

**Comparison to modern drivers**  
- Modern audio: ALSA (Linux), WASAPI (Windows), Core Audio (macOS) abstract hardware completely
- 1990s: Direct hardware access was necessary for latency-critical games
- Soundscape never achieved SoundBlaster's market dominance—hence its marginal presence in the codebase

## Potential Issues

- **ISR re-entrancy**: The stack allocation/deallocation functions hint at careful ISR management, but re-entrance bugs in interrupt handlers are notoriously subtle and hard to debug.
- **Hardware detection fragility**: `SOUNDSCAPE_FindCard()` likely probes memory-mapped I/O ranges; failure modes (wrong base address, missing hardware) may be silent without debug output.
- **DMA buffer alignment**: Some DMA controllers require page-aligned buffers; if not enforced in `SOUNDSCAPE_SetupDMABuffer`, memory corruption could occur.
- **Codec initialization order**: The gate-array and AD-1848 must be initialized in correct order; a race condition could lock up hardware.
