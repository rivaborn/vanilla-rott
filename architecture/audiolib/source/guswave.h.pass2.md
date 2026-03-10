# audiolib/source/guswave.h — Enhanced Analysis

## Architectural Role

GUSWAVE.H defines the public API for GUS (Gravis UltraSound) sampled audio playback. It is one of several hardware-specific audio modules (alongside ADLIBFX, BLASTER, AWE32, AL_MIDI) in a pluggable audio subsystem architecture. GUSWAVE encapsulates GUS-specific voice management, real-time pitch/pan control, and interrupt-driven sample playback, allowing the game to abstract away hardware details and support multiple sound devices via interchangeable modules.

## Key Cross-References

### Incoming (who depends on this file)
- Game engine audio manager (inferred from playback request patterns) — calls `GUSWAVE_Init`, `GUSWAVE_PlayVOC`/`PlayWAV`, `GUSWAVE_SetPan3D`, `GUSWAVE_Kill`, `GUSWAVE_Shutdown`
- Audio callback system (inferred from callback registration) — invokes registered function via `GUSWAVE_SetCallBack`
- Main game loop — likely queries `GUSWAVE_VoicesPlaying()`, `GUSWAVE_VoicePlaying(handle)` for state management
- Global volume control — via `GUSWAVE_SetVolume` and `GUSWAVE_GetVolume`

### Outgoing (what this file depends on)
- GUS hardware driver layer (implementation in `guswave.c`, not exposed in header)
- Watcom C runtime ABI (`#pragma aux` directives for frame setup)
- C standard library for error string return (char*) — likely using static string table

## Design Patterns & Rationale

**1. Voice Pool with Priority-Based Preemption**
The voice/handle abstraction (return value ≥ `GUSWAVE_MinVoiceHandle`) follows classic sound hardware management. `GUSWAVE_VoiceAvailable(priority)` gates voice allocation and enables voice stealing: if all voices are busy, lower-priority sounds can be interrupted. This was essential on 1990s hardware (GUS typically had 32 voices) and allowed rich audio without sophisticated mixing algorithms.

**2. Modular Audio Hardware Abstraction**
GUSWAVE mirrors the interface of ADLIBFX, BLASTER, and other modules, suggesting a **strategy pattern** for hardware drivers. The game code can switch modules or support multiple cards simultaneously without changing playback logic. Each module handles:
- Init/Shutdown lifecycle
- Format-specific playback (WAV, VOC)
- Real-time parameter control
- Completion callbacks
- Consistent error codes

**3. Interrupt-Driven Completion Callback**
`GUSWAVE_SetCallBack(function)` registers a global completion handler (likely invoked by GUS hardware interrupt, running at ISR level). This avoids polling and reduces latency, but imposes strict constraints: the callback function receives only a `callbackval` identifier and must not call unsafe functions (no malloc, no I/O, minimal state mutation).

**4. 3D Audio Positioning**
`GUSWAVE_SetPan3D(handle, angle, distance)` maps game-world spatial coordinates (azimuth 0–359°, distance) to stereo pan and volume attenuation. This was important for immersive FPS gameplay in the mid-1990s and avoids the need for a separate 3D audio library.

**5. Pull-Based Streaming (Demand-Feed)**
`GUSWAVE_StartDemandFeedPlayback()` uses a callback that the driver calls repeatedly to fetch audio chunks (`char **ptr, unsigned long *length`). This enables:
- Streaming long audio without pre-loading
- Procedurally generated audio
- Reduced memory footprint
- Real-time mixing on top of sampled data

## Data Flow Through This File

1. **Initialization Phase:**
   - Game calls `GUSWAVE_Init(numvoices)` → allocates voice pool, initializes GUS hardware
   - Optional: `GUSWAVE_SetCallBack(completion_fn)` registers ISR-level callback

2. **Playback Request:**
   - Audio manager calls `GUSWAVE_PlayVOC(sample, pitch, angle, vol, priority, callbackval)`
   - Returns voice handle (>= 1) or error code
   - GUS hardware begins fetching sample data from CPU memory, applying pitch/pan

3. **Real-Time Control (during playback):**
   - `GUSWAVE_SetPitch(handle, offset)` adjusts playback pitch (useful for Doppler, variable-speed effects)
   - `GUSWAVE_SetPan3D(handle, angle, distance)` updates spatial position (e.g., moving sound source)
   - `GUSWAVE_SetVolume(vol)` changes master volume (affects all voices)

4. **Completion & Cleanup:**
   - Voice finishes playing → GUS hardware signals interrupt → registered callback invoked with `callbackval`
   - Game can query `GUSWAVE_VoicePlaying(handle)` to check status
   - `GUSWAVE_Kill(handle)` or `GUSWAVE_KillAllVoices()` explicitly stops and frees voices

5. **Shutdown:**
   - `GUSWAVE_Shutdown()` halts all voices, releases GUS hardware resources

## Learning Notes

**Idiomatic to 1990s game audio:**
- **Handle-based voice management** rather than object references (common in systems programming of that era)
- **Format-explicit functions** (PlayVOC vs. PlayWAV) instead of a unified `Play(sample, format)` — reflects design priorities (minimize branching in hot paths)
- **Interrupt-level callbacks** for responsiveness; no event queues or thread safety (single-threaded DOS/Windows assumptions)
- **Raw pointer-based audio data** (no encapsulation) — caller is trusted to provide valid memory
- **Priority system** for voice contention — gamified audio mixing (important SFX always play)

**Modern engine differences:**
- Modern engines use **streaming decoders** (codec libraries) rather than format-specific loaders
- **Thread-based audio** rather than ISR callbacks (safer, more portable)
- **Entity component systems** where audio is a component; voices are managed implicitly
- **Spatial audio middleware** (e.g., Wwise, FMOD) rather than simple angle/distance panning

**Key engine concept:**
This file exemplifies **hardware abstraction layers** — a single interface for multiple hardware implementations. Essential for 1990s PC gaming where sound cards varied widely (Sound Blaster, GUS, AWE32, AdLib). Modern engines abstract higher (codec format → audio graph → platform mixer).

## Potential Issues

1. **Unsafe Audio Data Pointers:**  
   `GUSWAVE_PlayVOC`/`PlayWAV` take raw `char *sample` with no length or bounds checking. If the sample pointer is invalid or points to freed memory, undefined behavior occurs (likely hardware crash in 1990s).

2. **Callback ISR Context Assumptions:**  
   Registered callback via `GUSWAVE_SetCallBack` runs at interrupt level. No visible guard against unsafe operations (malloc, I/O, re-entrant calls). If game code violates ISR constraints, silent corruption or deadlock may result.

3. **Voice Handle Reuse:**  
   No visible validation that returned handles are still valid (e.g., if a voice is killed, can the handle be reused?). Caller must track handle lifecycle manually.

4. **Error Handling Granularity:**  
   Error enum includes `GUSWAVE_NoVoices` (all voices in use) but no obvious mechanism to wait or queue; caller must handle denial explicitly.

5. **Demand-Feed Buffer Management:**  
   `StartDemandFeedPlayback` callback signature (`void (*function)(char **ptr, unsigned long *length)`) requires caller to manage buffer lifecycle. No obvious safeguards against underruns or stale pointers.
