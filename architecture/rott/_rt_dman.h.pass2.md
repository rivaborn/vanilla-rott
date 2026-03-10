# rott/_rt_dman.h — Enhanced Analysis

## Architectural Role
This is a **private configuration header** for the digital audio manager subsystem. The prefix `_rt_` and guard name `_rt_dmand_private` indicate this is an internal constant set used by real-time audio I/O machinery, likely the intermediary between the high-level audio library (audiolib with Blaster, GUS, ADLib support) and the game's audio playback/recording pipelines. These constants directly configure DMA buffer allocation and streaming chunk sizes for interrupt-driven audio.

## Key Cross-References

### Incoming (who depends on this file)
- **Audio manager implementation** (likely `rott/_rt_dman.c` or similar) — consumes these buffer size constants during initialization
- **Game loop audio service routine** — likely references `PLAYBACKDELTASIZE` when servicing audio interrupts frame-by-frame
- **Recording/voice capture subsystem** — uses `RECORDINGSAMPLERATE` and `RECORDINGBUFFERSIZE` (e.g., voice chat in network play)

### Outgoing (what this file depends on)
- No dependencies; pure constants with no `#include` statements

## Design Patterns & Rationale
**Hardcoded configuration constants** — Rather than dynamic allocation or configuration files, buffer sizes are compile-time constants. This reflects 1990s game engine practice:
- Predictable memory layout (critical on DOS/early Windows)
- No runtime configuration overhead
- Sized to fit within conventional/extended memory limits

**Asymmetric buffer sizing** — Recording and playback buffers are equal (16 KB each), but playback has a separate "delta" size (256 bytes) for fine-grained streaming, suggesting a read-ahead or ring-buffer pattern where 256-byte chunks are consumed per interrupt/frame.

## Data Flow Through This File
1. **Initialization phase**: Audio manager allocates two 16 KB buffers (record, playback) and a small 256-byte working buffer
2. **Per-frame/per-interrupt**: Playback service reads 256-byte deltas from the playback buffer, advancing a position counter
3. **Recording phase** (if active): Audio input fills recording buffer at 7000 Hz sample rate; full 16 KB buffer consumed when complete
4. **Network play**: Voice capture likely routes through recording subsystem using these same constants

## Learning Notes
**What's idiomatic to this era:**
- **7 kHz sample rate** — Typical for voice recording in early 90s games (saves bandwidth in network play, acceptable for speech intelligibility)
- **Power-of-2 buffer sizes** — 16 KB = 2^14 bytes, optimized for DMA alignment and ring-buffer arithmetic
- **Separate delta size** — Suggests double-buffering or ring-buffer pattern; 256-byte chunks fit neatly into cache lines and enable fine control over DMA latency
- **No sample format specification** — Likely 8-bit mono (unsigned PCM), inferred from Blaster audio library support

**Modern contrast:** Contemporary engines use dynamic buffer sizing, runtime configuration, and sample-rate negotiation with the OS audio device.

## Potential Issues
- **Hard-coded 7 kHz** — Could be a bottleneck for voice quality in networked multiplayer; no abstraction for higher sample rates
- **No conditional compilation** — If `_rt_dman.h` is included in non-audio code paths, it pollutes the namespace with audio-specific symbols (minor issue given the naming convention)
- **Single buffer size for all audio types** — Recording and playback both use 16 KB regardless of sample rate or format, suggesting potential waste or contention if multiple subsystems share buffers
