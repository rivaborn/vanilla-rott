# audiolib/source/_midi.h — Enhanced Analysis

## Architectural Role

This header file defines the **MIDI protocol layer** of the audio subsystem—the abstraction boundary between Standard MIDI file parsing and hardware-specific sound drivers. It sits between file I/O (which reads `songcontext` and `track` state) and driver implementations (ADLIBFX, AWE32, AL_MIDI, etc.), which receive playback commands via a **function pointer callback table** (`_MIDI_Funcs`). The EMIDI (Extended MIDI) extensions allow hardware-specific behavior variants (context stacks, track filtering) to be encoded directly in MIDI files without coupling the parser to any single device.

## Key Cross-References

### Incoming
- **Task scheduler**: Calls `_MIDI_ServiceRoutine(task *Task)` at regular timer intervals (likely 70–140 Hz based on DOS-era convention)
- **Driver modules** (AL_MIDI, ADLIBFX, AWE32, BLASTER): Populate `_MIDI_Funcs` function table with hardware-specific implementations of `ProgramChange()`, `ControlChange()`, `NoteOn()`, `NoteOff()`
- **Configuration/initialization code**: Sets `_MIDI_Context`, `_MIDI_PatchMap` (instrument → hardware program mapping), and other globals
- **Public MIDI API** (likely `audiolib/source/midi.h`): Exposes song play/stop/pause, with `_midi.h` providing internal state management

### Outgoing
- **Driver function table callbacks**: `_MIDI_Funcs->ProgramChange(channel, program)`, `_MIDI_Funcs->ControlChange(channel, c1, c2)`, and others (not explicitly declared here, but referenced in commented code)
- **Global state shared with MIDI.C**: `_MIDI_Time`, `_MIDI_FPSecondsPerTick`, `_MIDI_Tick`, `_MIDI_Beat`, `_MIDI_Measure`, `_MIDI_BeatsPerMeasure`, `_MIDI_TicksPerBeat`, `_MIDI_TimeBase`, `_MIDI_Context`, `_MIDI_PatchMap`
- **Task type** (from system headers): Passed to service routine, suggesting integration with a cooperative task scheduler (likely `kernel.h` or DOS task management)

## Design Patterns & Rationale

1. **Function Pointer Dispatch Table** (`_MIDI_Funcs`)
   - Decouples MIDI file interpretation from hardware control, enabling driver swapping (Adlib ↔ AWE32 ↔ General MIDI) without recompiling the parser
   - Classic 1990s approach predating virtual functions and polymorphism in C

2. **Multi-Level Context Stack** (`songcontext` array)
   - Allows EMIDI to encode hardware-specific track variations (one context per supported device) without duplicating track data
   - Enables runtime hardware selection by switching `currentcontext` index

3. **Interrupt-Driven State Machine**
   - Playback advances one "tick" per `_MIDI_ServiceRoutine()` call, maintaining musical timing across system interrupts
   - Timing tracked in hierarchical units: tick → beat → measure (for synchronization with gameplay events)

4. **Big-Endian Binary Parsing**
   - `_MIDI_ReadNumber()` and `_MIDI_ReadDelta()` handle MIDI file's binary format (big-endian integer/variable-length encoding)
   - `RELATIVE_BEAT()` macro compresses 3D timing into a single scalar for comparison/sorting

## Data Flow Through This File

```
MIDI File (binary)
    ↓
_MIDI_ReadNumber() parses header & track metadata
_MIDI_ReadDelta()  parses variable-length timing deltas
    ↓
track.pos advances through event stream
    ↓
Running status + channel/command extracted via GET_MIDI_* macros
    ↓
Event dispatcher:
  - _MIDI_MetaEvent() → global timing updates (tempo, time sig)
  - _MIDI_SysEx() → system exclusive (vendor-specific)
  - _MIDI_InterpretControllerInfo() → channel messages (volume, program, pan)
    ↓
_MIDI_Funcs callback → Hardware driver (AL_ProgramChange, ADLIBFX_ControlChange, etc.)
    ↓
Hardware state updated; audio emitted
```

Looping and context switching: `songcontext.loopstart` and `loopcount` (from commented EMIDI loop code) allow tracks to loop back to saved state; `currentcontext` selects hardware-specific variant.

## Learning Notes

- **Interrupt-Driven Sequencing**: Pre-threading era (1994–95); playback driven by hardware timer or cooperative task yields. Modern engines use dedicated sequencer threads or event queues.
- **Callback-Based Hardware Abstraction**: No inheritance or polymorphism; drivers register a function table, enabling modular hardware support without C++ overhead.
- **EMIDI as a File-Level Variation System**: Rather than separate files per hardware target, Apogee embedded context stacks directly in MIDI files, allowing single asset to adapt to runtime hardware.
- **Bit-Shifting for Compact Encoding**: `RELATIVE_BEAT()` and `GET_MIDI_CHANNEL/COMMAND` macros show optimization for compact state representation (typical of memory-constrained DOS era).
- **Global State Coupling**: Timing and context globals (`_MIDI_Time`, `_MIDI_Context`) make threading or reentrancy impossible; modern design would encapsulate in a `MIDI_Engine` struct.

## Potential Issues

- **Commented Code (lines 199–272)**: Large block of EMIDI loop/context handling is commented out, suggesting either incomplete implementation or feature removal. If intentional, indicates design churn or hardware-specific features not fully deployed.
- **No Visible Bounds Checking**: While `_MIDI_ReadNumber()` and `_MIDI_ReadDelta()` are declared here, no buffer overflow protection is visible in this header. Real implementation (in MIDI.C) must validate file integrity.
- **Global State Tight Coupling**: Playback state scattered across module-level globals rather than encapsulated in a handle/instance. Makes concurrent playback (multiple songs) or clean shutdown difficult.
