# rott/rt_dmand.c — Enhanced Analysis

## Architectural Role
This file bridges the audio (FX) and network subsystems to enable bidirectional voice streaming in multiplayer sessions. It implements the demand-feed audio interface: recording captures local audio for network transmission, while playback handles incoming remote audio. The module sits at a critical convergence point where real-time audio timing (callback-driven) meets asynchronous network latency (chunk-based).

## Key Cross-References

### Incoming (who depends on this file)
- **Network layer** (`rt_net.c`): Calls `SD_StartIncomingSound`, `SD_UpdateIncomingSound`, `SD_StopIncomingSound`, and polls `SD_GetSoundData` to retrieve recorded chunks for transmission
- **Game logic** / main loop: Calls `SD_StartRecordingSound`, `SD_StopRecordingSound` to control mic input; checks `SD_SoundDataReady` before querying data
- **Cross-system coordination**: External subsystems check `SD_RecordingActive` (via semaphore) to determine if local recording is flagged active—decoupling intent from device state

### Outgoing (what this file depends on)
- **FX subsystem** (`fx_man.h`): Registers callbacks with `FX_StartDemandFeedPlayback` and `FX_StartRecording`; receives `Playingvoice` handle; calls `FX_StopSound`, `FX_StopRecord` to stop playback/recording
- **Memory management** (`rt_util.h`): `SafeMalloc`, `SafeFree` for circular buffers
- **Global state flags** (`rt_sound.h`): Reads `SD_Started`; (`rt_net.h`): Reads `remoteridicule` to gate recording start
- **Debug support** (`develop.h`): Writes `whereami` markers (69–74) in recording callback for crash diagnostics

## Design Patterns & Rationale

**Circular Buffers with Power-of-2 Sizing**: Both recording and playback use fixed-size circular buffers with wrap-around via `& (BUFFERSIZE - 1)`. This allows deterministic memory allocation critical for real-time audio—no dynamic allocation during playback/recording callbacks.

**Producer-Consumer via Callbacks**: Rather than polling, the FX subsystem pushes data via callbacks (`SD_UpdatePlaybackSound`, `SD_UpdateRecordingSound`). This decouples timing: audio hardware interrupts drive I/O independently of the main game loop. Network code then pulls recorded data at its own cadence via `SD_GetSoundData`.

**Sentinel-Based State Tracking**: Uses `-1` for uninitialized pointers (`FeederPointer=-1`, `PlayingPointer=-1`) to distinguish "not yet started" from "at position 0." This avoids an extra state variable.

**Semaphore-Based Intent Signaling**: The `RecordingSemaphore` pair (`SD_SetRecordingActive` / `SD_ClearRecordingActive`) separates "should we record?" from "are we recording?"—allowing other subsystems to express intent while device initialization is handled locally.

## Data Flow Through This File

**Recording Path**:
```
Mic Input (audio device)
  ↓ (FX interrupt)
SD_UpdateRecordingSound (callback)
  → writes to RecordingBuffer[RecordingPointer]
  → sets Feeder=true when first data arrives
  ↓ (main loop, async)
SD_GetSoundData (pull query)
  ← reads from RecordingBuffer[FeederPointer]
  ← returns rs_newsound (first chunk), rs_data (subsequent), rs_endsound (done)
  → Network layer transmits chunks
```

**Playback Path**:
```
Network layer (incoming chunks)
  ↓ (game loop)
SD_UpdateIncomingSound
  → writes to PlaybackBuffer[PlaybackPointer]
  ↓ (FX interrupt)
SD_UpdatePlaybackSound (callback, pulled by FX)
  → reads from PlaybackBuffer[PlayingPointer]
  → feeds audio to hardware
  ↓ (when done)
FX stops and module cleans up
```

State machine: `Recording` tracks hardware recording; `Feeder` tracks data availability; `Playing` vs. `Playback` distinguish actual playback from reception mode.

## Learning Notes

- **Circular Buffer Arithmetic**: The bitwise AND `& (SIZE - 1)` works only when SIZE is a power of 2—a hard constraint worth documenting. Modern engines often use modulo for clarity even if slower.
- **Callback Serialization Assumption**: The code assumes all callbacks from FX are serialized (single-threaded callback context) or at least non-concurrent with each other. No locks present—this was safe in DOS/Win95 (single-threaded audio interrupts) but fragile in modern multithreaded systems.
- **Voice Handle as Integer**: `Playingvoice` is an opaque handle (typecast to int). The code stores only one active voice at a time—no multiplexing of overlapping audio streams per session.
- **Deferred Cleanup**: Buffers are freed not immediately on `StopIncomingSound` but lazily when the playback callback detects completion (`PlayingPointer == PlaybackPointer`). This pattern avoids use-after-free if callbacks race with stop commands.
- **Debug Instrumentation (`whereami`)**: Scattering numbered markers through a callback function is a 1990s debugging technique—suggests this code was difficult to troubleshoot (likely race conditions or interrupt timing issues).

## Potential Issues

- **No Explicit Synchronization**: The code relies on FX callback discipline; if FX were ever to call `SD_UpdateRecordingSound` concurrently (future hardware, async drivers), buffer pointers could corrupt or data could be lost. Even single-threaded DOS code is vulnerable if interrupt handlers re-enter.
- **Silent Failure on Malloc**: `SafeMalloc` failure returns NULL and the functions set flags (`Playback=false`) without logging, making failures invisible unless caller checks return codes.
- **Hardcoded Buffer Sizes**: `PLAYBACKBUFFERSIZE`, `RECORDINGBUFFERSIZE`, `PLAYBACKDELTASIZE`, `RECORDINGSAMPLERATE` are #defined constants (not in this file, likely in headers). No runtime negotiation with network MTU or FX constraints.
- **Pointer Wrap-Around Edge Cases**: If exact wrap happens and `PlayingPointer == PlaybackPointer` during concurrent updates (hypothetically), the condition check for "buffer full" (`if (PlaybackPointer==PlayingPointer)`) might miss or falsely trigger depending on update order.
