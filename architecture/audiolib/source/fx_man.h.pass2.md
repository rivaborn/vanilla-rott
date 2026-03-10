# audiolib/source/fx_man.h — Enhanced Analysis

## Architectural Role
FX_MAN.H serves as the **unified sound effects abstraction layer** for the Apogee game engine, presenting a hardware-agnostic public API that shields the game from low-level sound card complexity. It acts as a facade over multiple hardware-specific drivers (BLASTER for Sound Blaster, ADLIBFX for AdLib FM synthesis, AL_MIDI for general MIDI, AWE32 for SoundBlaster AWE32). The header encapsulates voice management, sample playback, spatial audio, and effects processing, exposing a single entry point (`FX_Init`) that internally selects and initializes the appropriate driver based on detected hardware.

## Key Cross-References

### Incoming (who depends on this file)
- **Game engine / main loop**: Game code calls `FX_PlayVOC()`, `FX_PlayWAV()`, `FX_Pan3D()` for sound effects during gameplay events (footsteps, weapon fire, enemy ambient).
- **Menu system**: Likely calls `FX_PlayVOC()` for UI feedback sounds.
- **Cinematic/scripting system**: May use `FX_StartDemandFeedPlayback()` for streamed music during cutscenes.
- **Configuration/settings system**: Calls `FX_SetVolume()`, `FX_SetReverb()` to apply user audio preferences.

### Outgoing (what this file depends on)
- **`sndcards.h`**: Provides sound card type enumeration (referenced but not expanded in provided context).
- **BLASTER driver** (`audiolib/source/blaster.h`, `blaster.c`): Low-level Sound Blaster I/O, DMA setup, ISR handling. FX_MAN likely calls `BLASTER_Init()`, `BLASTER_BeginBufferedPlayback()`, `BLASTER_SetCallBack()`.
- **ADLIBFX driver** (`audiolib/source/adlibfx.h`, `adlibfx.c`): FM synthesis backend for AdLib cards (alternative hardware support).
- **AL_MIDI driver** (`audiolib/source/al_midi.h`, `al_midi.c`): MIDI synthesis for FM cards or external synths.
- **AWE32 driver** (`audiolib/source/awe32.h`, `awe32.c`): SoundBlaster AWE32 wavetable synthesis backend.
- **DMA module** (`audiolib/source/dma.h`, `dma.c`): Direct Memory Access setup for efficient sample transfer (likely called indirectly via BLASTER).
- **Voice pool**: Internal state machine tracks active voice handles, priorities, and callbacks.

## Design Patterns & Rationale

**Facade Pattern**: FX_MAN presents a simple, unified interface (`FX_Init`, `FX_PlayVOC`, etc.) while hiding hardware selection, device initialization, and driver-specific details. Game code remains oblivious to whether a Sound Blaster 16, AdLib, or AWE32 is present.

**Callback-Based Completion Notification**: Sound completion (`callbackval` parameter) uses function pointers rather than polling. This is idiomatic for 1990s DOS/ISR-driven engines where blocking waits are unacceptable.

**Priority-Based Voice Stealing**: The `priority` parameter in playback functions enables intelligent voice allocation: higher-priority sounds (e.g., critical feedback) evict lower-priority sounds (e.g., ambient) when voices exhaust. This is essential given limited voice budgets (4–8 voices typical on Sound Blaster).

**Handle-Based Resource Management**: Playback functions return opaque integer handles (e.g., from `FX_PlayVOC()`), allowing game code to manage active sounds without direct voice struct access. Handles decouple the public API from internal voice representation.

**Dual API for Spatial Audio**: Both explicit stereo panning (`FX_SetPan`, `left`/`right` parameters) and 3D spatial variants (`FX_PlayVOC3D`, `FX_Pan3D`) coexist. The 3D variants internally convert angle/distance to pan/volume, providing high-level abstraction for game developers unfamiliar with audio math.

## Data Flow Through This File

1. **Initialization Phase**: Game calls `FX_Init(SoundCard, numvoices, ...)` → internally calls `FX_SetupCard()` to detect capabilities, `FX_GetBlasterSettings()` to read hardware config (from BLASTER env var or hardware probe), then calls hardware-specific setup (e.g., `FX_SetupSoundBlaster()`) → ISR/DMA initialized, voice pool allocated.

2. **Gameplay Audio Emission**: On game event (player fires weapon):
   - Game calls `FX_PlayVOC(sound_ptr, pitch, vol, left, right, priority, callback_id)`
   - FX_MAN allocates a voice (steals if needed), starts DMA/ISR playback via BLASTER driver
   - Returns handle to game
   - Game stores handle if dynamic control needed (e.g., `FX_Pan3D(handle, angle, distance)` for moving sound source)

3. **Completion Callback**: When DSP/DMA finishes sample playback:
   - ISR fires, calls registered callback from `FX_SetCallBack()`
   - Callback invokes game's sound-completion handler with stored `callback_id`
   - Game can queue next sound or free UI resources

4. **Shutdown**: Game calls `FX_Shutdown()` → stops all sounds, disables ISR, releases voice buffers, resets hardware.

## Learning Notes

**What Modern Engines Do Differently**:
- Modern engines use **unified audio graph/mixer architecture** (e.g., Web Audio API, FMOD, Wwise) where each sound is a source node with properties updated per-frame, rather than ISR-driven callbacks.
- **Dynamic 3D audio** is now computed in the mixer thread (not ISR), allowing smoother interpolation and more sophisticated spatial algorithms (HRTF, room simulation).
- **Memory management**: Modern engines pre-allocate or stream samples on-demand; this 1994 engine expects samples already loaded in RAM.

**Idiomatic to This Engine**:
- **Interrupt-Service-Routine (ISR) driven audio**: The callback mechanism and ISR-based voice lifecycle reflect DOS real-mode constraints where the main game loop and audio mixer are tightly coupled.
- **VOC/WAV/RAW format handling in userland**: No OS-level audio API (DirectSound didn't exist yet); sound card is directly programmed.
- **Reverb as post-process**: Reverb parameters suggest a simple feedback-delay network (FDN) or convolution, computed on the mixed output, not per-voice.
- **Looping by sample offset**: Looping is defined by byte offsets (loopstart/loopend), requiring the loader to pre-calculate loop boundaries rather than relying on metadata tags.

**Game Engine Concepts**:
- Relates to **voice pooling** and **priority arbitration** (resource scarcity under constraints).
- Approximates **spatial audio** via simple stereo panning heuristics (not true 3D convolution).
- Demonstrates **callback-driven event systems** for async completion notification (precursor to modern event queues/async/await).

## Potential Issues

1. **No explicit thread safety**: If the game loop calls `FX_PlayVOC()` while ISR fires a callback simultaneously, race conditions could corrupt voice state. Likely mitigated by disabling interrupts during critical sections in fx_man.c, but not evident from this header.

2. **Callback Parameter Passing**: The `callbackval` parameter is the only way to pass context to completion callbacks. Games must encode state into a single `unsigned long`, limiting extensibility (e.g., can't pass pointers in 16-bit DOS real mode; workarounds using handle tables required).

3. **Hardcoded Sound Blaster Support**: The enum `fx_BLASTER_Types` and functions like `FX_SetupSoundBlaster` suggest Sound Blaster is the primary target. AdLib, AWE32, and MIDI support may have been added later; initialization logic for selecting the "active" driver is not visible in this header, implying complex conditional logic in fx_man.c.

4. **No Resource Cleanup on Error**: If `FX_Init()` fails mid-initialization, callers have no way to partial-cleanup. The header doesn't expose low-level driver handles or state, making error diagnosis difficult.
