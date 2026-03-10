# audiolib/source/_midi.h

## File Purpose
Private C header file for MIDI song playback engine. Defines constants, message formats, and data structures for Standard MIDI file parsing and Extended MIDI (EMIDI) hardware-specific playback support. Part of the audio library's music playback subsystem authored by James R. Dose (Apogee Software, 1994–1995).

## Core Responsibilities
- Define MIDI protocol constants: message types, control codes, meta-event codes, hardware signatures
- Define playback state structures for tracks and song contexts with looping/timing metadata
- Declare private functions for MIDI event processing, timing, and hardware control
- Support EMIDI extensions for hardware-specific behavior (Adlib, Sound Canvas, etc.)
- Provide bit-manipulation and delta-time parsing helpers

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `songcontext` | struct | Per-context playback state: track position, loop counters, running status, timing/tempo data, measure/beat/tick tracking |
| `track` | struct | MIDI track wrapper holding multiple song contexts, EMIDI per-track settings (volume, program change handling), active context index |

## Global / File-Static State
None (declared here; definitions in MIDI.C).  
*Note: Commented code references global state: `_MIDI_Time`, `_MIDI_FPSecondsPerTick`, `_MIDI_Tick`, `_MIDI_Beat`, `_MIDI_Measure`, `_MIDI_BeatsPerMeasure`, `_MIDI_TicksPerBeat`, `_MIDI_TimeBase`, `_MIDI_Context`, `_MIDI_Funcs` (function table), `_MIDI_PatchMap` (instrument mapping).*

## Key Functions / Methods

### _MIDI_ReadNumber
- **Signature:** `static long _MIDI_ReadNumber(void *from, size_t size)`
- **Purpose:** Parse multi-byte binary integer from MIDI file buffer (big-endian).
- **Inputs:** Pointer to buffer, byte count.
- **Outputs/Return:** Parsed integer value.
- **Calls:** (none visible)

### _MIDI_ReadDelta
- **Signature:** `static long _MIDI_ReadDelta(track *ptr)`
- **Purpose:** Parse variable-length delta time from track buffer.
- **Inputs:** Track pointer (contains `pos` iterator).
- **Outputs/Return:** Delta time in ticks.
- **Side effects:** Advances `ptr->pos`.

### _MIDI_ServiceRoutine
- **Signature:** `void _MIDI_ServiceRoutine(task *Task)` (declared but definition elsewhere)
- **Purpose:** Main playback callback; processes one tick of MIDI events.
- **Inputs:** Task control block.
- **Side effects:** Advances playback state, sends control changes, updates global timing.
- **Notes:** Likely invoked by a task/timer scheduler at fixed tick intervals.

### _MIDI_MetaEvent, _MIDI_SysEx
- **Purpose:** Process MIDI meta-events (tempo, time signature, etc.) and system exclusive messages.
- **Inputs:** Track pointer.
- **Side effects:** Update track state or global timing.

### _MIDI_InterpretControllerInfo
- **Signature:** `static int _MIDI_InterpretControllerInfo(track *Track, int TimeSet, int channel, int c1, int c2)`
- **Purpose:** Dispatch MIDI control messages (volume, pan, EMIDI extensions).
- **Inputs:** Track, time-set flag, MIDI channel, controller number (c1), value (c2).
- **Notes:** Commented code shows handling for EMIDI loop markers, volume changes, program changes.

### _MIDI_SetChannelVolume, _MIDI_SendChannelVolumes
- **Purpose:** Set per-channel volume and apply global volume updates.

**Notes on remaining functions:**
- `_MIDI_ResetTracks`, `_MIDI_AdvanceTick`, `_MIDI_SendControlChange`, `_MIDI_ProcessNextTick`, `_MIDI_InitEMIDI` are straightforward per-step playback operations.

## Control Flow Notes
Integration point: Per-frame/tick in main playback loop. `_MIDI_ServiceRoutine` is likely called from a task scheduler or timer interrupt to advance playback state and emit MIDI messages (via `_MIDI_Funcs` function table) to the sound driver. Timing tracked in ticks → beats → measures using `TimeBase` and `TicksPerBeat`. Looping handled via `songcontext` stack with loop counters and saved playback state.

## External Dependencies
- **Global state (defined elsewhere):** `_MIDI_Time`, `_MIDI_FPSecondsPerTick`, `_MIDI_Tick`, `_MIDI_Beat`, `_MIDI_Measure`, `_MIDI_Context`, `_MIDI_Funcs`, `_MIDI_PatchMap`
- **Type:** `task` (appears in service routine signature; likely OS/scheduler task type)
- **Implicit:** Sound driver interface via `_MIDI_Funcs->ProgramChange()`, `_MIDI_Funcs->ControlChange()` callbacks
