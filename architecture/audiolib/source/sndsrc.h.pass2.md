# audiolib/source/sndsrc.h — Enhanced Analysis

## Architectural Role
SoundSource driver is one of several hardware-specific audio device drivers in the audiolib subsystem, positioned alongside BLASTER, ADLIBFX, AWE32, and GUS drivers (visible in cross-references). It implements a device abstraction layer for legacy parallel-port audio hardware, allowing the engine to support multiple audio devices through a consistent callback-driven DMA interface. The modular design suggests a device-detection and routing layer (likely in a higher-level audiolib file) selects which driver to initialize at runtime.

## Key Cross-References

### Incoming (who depends on this)
- Not directly visible in cross-reference excerpt, but the identical function signatures across BLASTER, ADLIBFX, and other drivers (e.g., `_Init`, `_Shutdown`, `_BeginBufferedPlayback`, `_SetCallBack`, `_SetMixMode`) strongly implies a higher-level audiolib dispatcher calls these functions polymorphically after device detection.
- Callback mechanism suggests integration with an interrupt-driven audio service routine (likely in `audiolib/source/` core).

### Outgoing (what this file depends on)
- **Hardware I/O**: Direct parallel port writes (0x3bc, 0x278, 0x378)
- **DPMI subsystem**: `SS_LockMemory`/`SS_UnlockMemory` rely on DOS protected-mode memory management
- **No cross-audiolib calls**: Unlike some device drivers (e.g., GUS MIDI), SoundSource is audio-output-only; no dependency on sequencer or MIDI modules

## Design Patterns & Rationale

**Device Driver Abstraction**  
API surface mirrors other audiolib drivers, enabling compile-time or runtime polymorphism. This reduces coupling between game logic and hardware-specific code.

**Ring Buffer + Callback Pattern**  
`NumDivisions` in `SS_BeginBufferedPlayback` divides the buffer into segments, triggering the callback when each division is consumed by DMA. This is the canonical DOS-era streaming pattern—avoids busy-wait, integrates naturally with interrupt-driven I/O.

**DPMI Memory Locking**  
`SS_LockMemory` / `SS_UnlockMemory` bracket playback operations, ensuring DMA-addressed buffers don't get paged to disk. Reflects that this code targets 32-bit DOS extenders (DPMI), not real-mode DOS. Shows awareness of protected-mode pitfalls.

**Port Selection Constants**  
Three standard parallel port bases (0x3bc, 0x278, 0x378) + Tandy variant allow hardware flexibility without recompilation—users select via compile-time define or runtime `SS_SetPort()`.

## Data Flow Through This File

**Initialization Path:**
```
Game startup → audiolib device detection → SS_Init(soundcard) 
  → SS_SetPort(port) [if non-default]
  → SS_LockMemory()
```

**Playback Path:**
```
Audio buffer ready → SS_BeginBufferedPlayback(buf, size, divisions, callback)
  → [DMA active, interrupt fires per division]
  → Callback → Application refills ring buffer
  → SS_StopPlayback() → SS_UnlockMemory() → SS_Shutdown()
```

**State transitions**:
- Uninitialized → Initialized (SS_Init)
- Idle → Playing (SS_BeginBufferedPlayback)
- Playing → Stopped (SS_StopPlayback)
- Shutdown → Uninitialized (SS_Shutdown)

## Learning Notes

**1990s DOS Audio Paradigm:**  
SoundSource represents a specific era: parallel-port audio adapters (cheaper than Blaster, required no ISA slot). Modern engines use OS-level audio APIs; studying this shows how hardware abstraction worked in resource-constrained environments.

**Ring Buffer + Callback:**  
Still used in modern audio engines (e.g., low-latency audio on mobile), but via platform audio APIs. Developers studying RoTT can see the pattern implemented raw.

**DPMI Awareness:**  
`SS_LockMemory` is invisible to modern coders but essential here. It reveals that even by 1994, DOS gaming required protected-mode tricks to manage memory safely. The pattern appears in GUS, BLASTER drivers too—systemic to audiolib's design.

**Polymorphic Device Drivers:**  
No vtable-style code in header, but the identical API signatures across drivers strongly suggest the containing `.c` file uses function pointers or a device-selector macro. A developer wanting to add a new audio device would copy this header, implement the functions, and register with the dispatcher.

## Potential Issues

- **Hardware dependency**: Code assumes direct parallel port access—incompatible with modern OSes and USB adapters. Not an issue for the original game but would require abstraction if ported.
- **Fixed sample rate (7000 Hz)**: Defined as constant; query functions (`SS_GetPlaybackRate`) suggest variable rates elsewhere in implementation, potential mismatch.
- **Callback context**: No documentation on callback thread/context safety—may assume single-threaded DOS, could be fragile if audiolib services multiple drivers concurrently.
