# audiolib/source/adlibfx.h — Enhanced Analysis

## Architectural Role

ADLIBFX is a **hardware-specific audio backend** for Adlib/FM synthesis sound effects within a pluggable audio architecture. It occupies the same subsystem tier as BLASTER (PCM), AL_MIDI (synthesizer), and AWE32 (wave table), suggesting a **modular driver pattern** where the engine selects an audio backend at runtime. This header defines the public contract for FM synthesis voice management—a legacy but crucial capability in 1990s DOS gaming.

## Key Cross-References

### Incoming (who depends on this file)
- **Upper-layer audio coordinator** (not visible in this excerpt, likely in `audiolib/source/` or core engine)  
  - The callback mechanism (`ADLIBFX_SetCallBack`) and voice availability queries (`ADLIBFX_VoiceAvailable`) suggest a dispatcher that routes sound effects to available hardware
  - Mirrors the same init/shutdown pattern as BLASTER, AL_MIDI, AWE32 → indicates runtime selection or fallback chaining

### Outgoing (what this file depends on)
- **Hardware/DPMI layer**: Memory locking primitives (`PCFX_LockMemory`, `PCFX_UnlockMemory`) imply direct ISR/DMA coordination  
  - No `#include` statements visible here, but implementation (adlibfx.c) must invoke low-level port I/O and interrupt handlers
- **No visible dependency on other AUDIOLIB subsystems** (no AL_MIDI, BLASTER includes)
  - Suggests **isolation by design**: each backend is self-contained and swappable

## Design Patterns & Rationale

**Pattern: Hardware Abstraction Layer (HAL) via Functional Interface**
- Public API is purely functional (no opaque structs or virtual method tables)
- Voice/handle abstraction hides FM operator complexity from callers
- `ALSound` struct carries raw FM operator parameters (attack, sustain, wave, modulation scaling) → suggests the **data-driven synthesis approach** where sound effects are pre-authored binary blobs, not computed at runtime

**Pattern: Interrupt-Safe Callback**
- Single global callback (`ADLIBFX_SetCallBack`) invoked when voices expire
- Passed `callbackval` (client context) allows sound effects to notify higher layers without exposing ISR details
- This contrasts with polled status queries (`ADLIBFX_SoundPlaying`), suggesting **hybrid event/polling** model

**Pattern: Priority-Based Voice Preemption**
- `ADLIBFX_VoiceAvailable(priority)` and `ADLIBFX_Play(..., priority, ...)` indicate **voice stealing** under resource contention
- Typical Adlib has 9 FM channels; priority ensures UI/critical sounds (gunfire, pickup) survive over background drones

## Data Flow Through This File

1. **Init phase**: `ADLIBFX_Init()` → `PCFX_LockMemory()` sets up ISR-safe memory and hardware state
2. **Sound playback request**: Caller loads `ALSound` struct (FM operator data), calls `ADLIBFX_Play(...)`
   - Returns voice handle; triggers hardware programming (operator envelopes, etc.)
3. **Playback**: ISR/service routine drives FM synthesis; voices decay per envelope
4. **Completion**: ISR invokes registered callback with `callbackval` → caller updates game state (removes effect, plays next sound)
5. **Stop/Shutdown**: Explicit `ADLIBFX_Stop(handle)` or `ADLIBFX_Shutdown()` releases voices, disables hardware

## Learning Notes

**1990s Audio Abstraction Philosophy:**
- No runtime polymorphism (no vtables) — static dispatch via module selection at build/init time
- Callback-driven completion notification is more efficient than polling 9 voices per frame
- Memory locking is a DOS-specific artifact (no virtual memory on Adlib hardware); modern engines use DMA-safe allocators

**FM Synthesis Idiom:**
- `ALSound` struct is essentially a **patch**: operator pairs (modulator + carrier) with ADSR envelopes and waveform selection
- The `mChar`/`cChar` fields are operator parameters; `nConn` is the modulation topology
- Pre-authored sound data (not real-time parameter tweaking) suggests sounds were designed offline in FM synthesis tools

**Comparison to Modern Engines:**
- Modern audio engines (FMOD, Wwise) abstract hardware via abstraction layers; this codebase simply has **isolated, swappable backends**
- No resource management abstraction (no event queue, no voice pool manager) → responsibility falls on caller
- The persistent callback pattern is ancestor to modern **completion events** in audio middleware

## Potential Issues

1. **Single global callback limitation**: If multiple concurrent sounds need different callbacks, only one can be registered. Workaround likely in higher-level dispatcher (not visible here).
2. **No error recovery details**: `ADLIBFX_Errors` enum doesn't specify whether DPMI failures, hardware hangs, or ISR conflicts are recoverable.
3. **Memory layout fragility**: `PCFX_Lock/UnlockMemory` requires exact knowledge of code/data footprint; if `adlibfx.c` allocation changes, ISR corruption is possible (typical late-1990s hazard).
