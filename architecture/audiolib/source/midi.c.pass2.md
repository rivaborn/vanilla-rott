# audiolib/source/midi.c — Enhanced Analysis

## Architectural Role

This file implements the core MIDI playback engine for the Apogee audio library, serving as the intermediary between MIDI song files and hardware synthesizers. It parses MIDI file structures, maintains tick-driven playback state (via the task manager), dispatches events to sound device drivers (accessed through device-specific callback tables), and manages playback lifecycle (load → play → seek → stop). The module supports both standard MIDI and EMIDI extensions (looping, context switching, track filtering), enabling sophisticated music sequencing for a DOS-era 3D game engine.

## Key Cross-References

### Incoming (who depends on this file)
- **Game engine code** (`rott/`) calls MIDI_PlaySong, MIDI_StopSong, MIDI_SetVolume, MIDI_GetSongPosition for music playback control and UI updates
- **Sound device drivers** (audio/source/al_midi.c, awe32.c, adlibfx.c) are invoked via `_MIDI_Funcs` callback table for NoteOn, NoteOff, ControlChange, ProgramChange, SetVolume, LoadPatch
- **Task scheduler** (task_man.h) triggers `_MIDI_ServiceRoutine` at ~100 Hz to process pending MIDI events

### Outgoing (what this file depends on)
- **Task manager** (task_man.h): TS_ScheduleTask, TS_Terminate, TS_SetTaskRate, TS_Dispatch for timer-driven playback
- **Memory allocation** (usrhooks.h): USRHOOKS_GetMem/FreeMem for track buffer allocation
- **Memory locking** (dpmi.h): DPMI_LockMemory/UnlockMemory for DOS protected-mode interrupt safety
- **Device drivers** (via midifuncs callbacks): device-specific NoteOn/NoteOff/ControlChange/ProgramChange/SetVolume/LoadPatch/Reset/ReleasePatches implementations
- **Sound card enumeration** (sndcards.h): MUSIC_SoundDevice global to determine active hardware (GenMidi, AWE32, UltraSound, etc.)
- **Interrupt control** (interrup.h): DisableInterrupts/RestoreInterrupts for timing-critical MIDI_Reset
- **EMIDI feature check** (via EMIDI_AffectsCurrentCard(c2, type)): conditional feature enablement based on hardware type

## Design Patterns & Rationale

**1. Task-Driven Playback Loop**
- Delegates event processing to task manager (~100 Hz) rather than main game loop; avoids blocking rendering
- `_MIDI_ServiceRoutine` is repeatedly invoked to advance tick counters and execute events with zero delay
- Rationale: DOS multitasking required cooperative, interrupt-safe task scheduling; allows responsive UI during music playback

**2. Device-Abstraction via Callback Table**
- `midifuncs` function pointers decouple MIDI playback logic from hardware specifics (GenMidi, Adlib, Ensoniq, AWE32)
- `_MIDI_RerouteFunctions[16]` per-channel interception allows user code to filter/transform MIDI events before device output
- Rationale: Single MIDI playback engine supports multiple hardware targets without conditional compilation

**3. Two-Tier Volume Control**
- `_MIDI_TotalVolume` (master) × `_MIDI_ChannelVolume[ch]` (set by control changes) × `_MIDI_UserChannelVolume[ch]` (user override)
- Decoupling allows independent master volume control and per-channel muting without re-parsing MIDI file
- Rationale: Game UI can adjust master volume and selectively mute/unmute channels; MIDI file retains original mix intent

**4. Fixed-Point Arithmetic for Timing**
- `_MIDI_FPSecondsPerTick` (left-shift by TIME_PRECISION bits) avoids floating-point in interrupt context
- Accumulated `_MIDI_Time` provides millisecond-accurate position for seek/resume across variable tempos
- Rationale: DOS era lacked hardware FPU; fixed-point avoids exception penalties and maintains determinism

**5. EMIDI Context Stack**
- Per-track context[0] stores save/restore points for loop boundaries, running status, and full playback state
- Context switching (EMIDI_CONTEXT_END) replays from saved state without re-scanning MIDI file
- Rationale: Enables seamless looping without file re-parsing; supports multi-pattern compositions and dynamic arrangements

**6. Variable-Length Quantity Decoding**
- `_MIDI_ReadDelta` implements VLQ (7-bit continuation, 1-bit flag per byte) for compact delta-time encoding
- Rationale: MIDI standard; saves space in multi-track files while maintaining stream synchronization

## Data Flow Through This File

```
MIDI File (in memory)
  ↓
MIDI_PlaySong: Validate header (MThd), enumerate tracks, allocate track[] buffer, lock memory
  ↓
_MIDI_InitEMIDI: Scan all events to measure song length, detect EMIDI features, populate context stacks
  ↓
MIDI_LoadTimbres: Pre-cache unique instrument patches on devices supporting LoadPatch
  ↓
MIDI_Reset: Initialize device to GM defaults (pitch bend sensitivity, default volumes)
  ↓
TS_ScheduleTask → _MIDI_ServiceRoutine (~100 Hz interrupt):
  ├─ For each track with delay==0:
  │  ├─ Read variable-length delta time (VLQ)
  │  ├─ Read MIDI event byte (with running status)
  │  ├─ Interpret: NoteOn/NoteOff → device NoteOn/NoteOff
  │  │             ControlChange → _MIDI_InterpretControllerInfo (volume, pan, EMIDI looping/context)
  │  │             ProgramChange → device ProgramChange
  │  │             MetaEvent → tempo/time-signature update
  │  └─ Check reroute callbacks (_MIDI_RerouteFunctions[ch]) before device output
  ├─ Decrement all track delays
  ├─ Advance global tick, beat, measure counters
  └─ If all tracks inactive: MIDI_Loop ? reset : stop
  ↓
Device output (hardware synth playback)
```

**Seeking**: Pause → Reset all tracks to start → Fast-forward via `_MIDI_ProcessNextTick` loop → Resume

## Learning Notes

**Idiomatic to Early Game Audio (1990s DOS)**
- **Tick-based event scheduling** rather than sample-accurate timing; ~100 Hz cadence is typical for 386/486-era CPUs
- **Global state vs. objects**: Single static track array, no encapsulation; reflects C procedural style and DOS single-threaded model
- **Memory locking for interrupt safety**: DPMI calls to lock code/data pages prevent page-faults in real-mode ISR context (modern OSes abstract this)
- **Hardware abstraction via callbacks**: Pre-dates modern plugin architectures; forces explicit coupling at compile time

**Modern Game Engines Differ**
- **Sample-accurate scheduling**: Real-time audio threads with millisecond or sub-millisecond precision
- **Streaming vs. in-memory**: Modern engines stream MIDI from disk; Apogee loads entire file into locked DOS memory
- **Standard MIDI sequencers** (like FMOD, Wwise): Abstract away hardware differences; support multiple simultaneous music tracks
- **ECS-like state**: Entity-component systems manage playback state; MIDI here is monolithic global state

**Deep Concept: EMIDI as Branching Composition**
- Context stacks enable non-linear playback: same MIDI file can play different "arrangements" based on context switches
- Loop points with variable counts (EMIDI_INFINITE) allow dynamic composition (e.g., boss battle music that extends until boss defeated)
- More sophisticated than traditional MIDI; foreshadows interactive music systems in modern games

## Potential Issues

1. **Fixed Task Rate (~100 Hz)**: Tempo changes (MetaEvent MIDI_TEMPO_CHANGE) recalculate TicksPerSecond via `TS_SetTaskRate`, but coarse granularity may accumulate timing drift over long songs. Modern audio engines use sample-accurate clocks.

2. **Backward Seek Inefficiency**: MIDI_SetSongTick/Time/Position always resets and fast-forwards; expensive for large files or frequent scrubbing. No indexing of event positions.

3. **EMIDI Context Complexity**: Context switching saves full playback state (20+ fields per track); risk of inconsistency if state updates bypass context save logic. Hard to audit correctness.

4. **No Thread Safety**: Global state (_MIDI_SongActive, _MIDI_PositionInTicks, _MIDI_Time) is accessed from task routine and main code without locks. Assumes single-threaded DOS environment; unsafe if called from multiple contexts.

5. **Device Callback Coupling**: If a device driver callback (e.g., _MIDI_Funcs->NoteOn) blocks or takes excessive time, it delays all subsequent tracks in that tick, causing audio glitching.
