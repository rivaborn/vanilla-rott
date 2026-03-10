# audiolib/source/gusmidi2.c — Enhanced Analysis

## Architectural Role

This module is the **GUS-specific MIDI driver implementation**, sitting between a higher-level music sequencer and the low-level GUS hardware abstraction (`gf1_*` functions). It converts MIDI events into GUS synthesis commands, manages patch (instrument) persistence, and enforces reference-counted hardware lifecycle. Within the multi-driver audio architecture (which also includes ADLIB, BLASTER, and AWE32 variants), `gusmidi2.c` provides the General MIDI interface for Gravis Ultrasound cards specifically.

## Key Cross-References

### Incoming (who depends on this file)
- **Unknown direct callers** in cross-reference list (likely a higher-level MIDI sequencer or music playback system in `rott/` that dispatches to MIDI drivers based on hardware selection)
- The public API (`GUSMIDI_Init`, `GUSMIDI_NoteOn`, `GUSMIDI_ControlChange`, etc.) is exposed via `gusmidi2.h` for use by game engine music playback layer

### Outgoing (what this file depends on)
- **GUS hardware layer** (`gf1_*` functions): `gf1_load_os()`, `gf1_unload_os()`, `gf1_load_patch()`, `gf1_unload_patch()`, `gf1_mem_avail()`, `gf1_midi_note_on()`, `gf1_midi_note_off()`, `gf1_midi_change_program()`, `gf1_midi_parameter()`, `gf1_midi_pitch_bend()`, `gf1_midi_master_volume()`
- **Configuration utilities** (`GetUltraCfg()` in `audiolib/source/gus.c`) — reads hardware setup from ULTRASND.INI
- **Shared utility** (`D32DosMemAlloc()` also defined in `gus.c`, `irq.c`) — memory allocation utility exported to other audio drivers
- **Standard I/O & memory** — `fopen()`, `fgets()`, `sscanf()`, `malloc()`, `free()`, DOS/DPMI services

## Design Patterns & Rationale

1. **Reference-counted hardware initialization** (`GUS_Installed` counter):
   - Multiple subsystems can safely initialize/shutdown GUS without trampling each other
   - Typical for shared hardware resources in a DOS/DPMI multitasking environment
   - True hardware load (`gf1_load_os()`) happens only on first init; true unload only on final shutdown

2. **Preload-all patch strategy**:
   - All 256 patches (128 melodic + 128 drum) loaded into GUS DRAM at init time
   - Trades upfront load time and memory for zero-latency note-on latency during gameplay
   - Acceptable in 1995 for General MIDI (fixed patch set)
   - Contrasts with dynamic loading used in other engines

3. **Memory-config-aware patch mapping**:
   - `patch_map[prognum][config]` allows same MIDI program to map to different patches based on available GUS DRAM (256K–1MB configurations)
   - Graceful degradation: smaller DRAM → fewer/lower-quality patches
   - Selected at init time (`GUS_MemConfig`) and never changes during runtime

4. **Interrupt-atomic unload** (disable/enable around patch unload):
   - Prevents concurrent access to patch data while unloading
   - Zeroes `nlayers` to immediately silence active notes (no explicit note-off needed)
   - Necessary because GUS synthesis may be running in interrupt context (DMA/timer-driven)

## Data Flow Through This File

**Initialization:**
```
GUSMIDI_Init() 
  → GUS_Init() [ref-count check]
    → GetUltraCfg() [read hardware config]
    → gf1_load_os() [load GUS driver]
  → D32DosMemAlloc(2048) [allocate DMA scratch buffer for patch loading]
  → GUS_GetPatchMap() [parse ULTRAMID.INI → populate patch_map[][] and program_name[][]]
  → Loop: GUSMIDI_LoadPatch(0..255) [load each patch into GUS DRAM]
```

**Runtime MIDI event dispatch:**
```
GUSMIDI_NoteOn(chan, note, vel)
  → patch index = patch_map[note+128 if chan==9, else 0][GUS_MemConfig]
  → gf1_midi_note_on(patch, chan, note, vel)

GUSMIDI_ProgramChange(chan, prog)
  → patch index = patch_map[prog][GUS_MemConfig]
  → gf1_midi_change_program(patch, chan)
  [similar for NoteOff, ControlChange, PitchBend, Volume]
```

**Shutdown:**
```
GUSMIDI_Shutdown()
  → GUSMIDI_ReleasePatches() [unload all 256 patches, free waveform buffers]
  → GUS_Shutdown() [decrement refcount, unload driver if refcount == 0]
```

## Learning Notes

**Idiomatic to this era (early 1990s game engines):**
- **DOS/DPMI reality**: FAR pointers, memory segmentation, interrupt management, conventional memory constraints (64K DMA buffer limit, 640K overall DOS heap)
- **Static preload strategy**: Modern engines lazy-load resources on demand or use streaming; this engine trades memory for predictable latency
- **Single memory config per run**: No hot-swapping of quality settings; init picks one patch set and sticks with it
- **INI file configuration**: Not XML/JSON; simple key=value parsing (no error handling for malformed files)
- **Polling pattern for error checking**: Explicit error code + auxiliary error pair (cf. `errno`); no exceptions

**Relationship to engine architecture:**
- Part of a pluggable driver system (ADLIB, BLASTER, AWE32, etc. in `audiolib/source/`)
- MIDI dispatch layer sits *below* a hypothetical sequencer that reads .MID files or sends note data
- Reference counting allows multiple game subsystems to use GUS without coordination
- Configuration baked into INI file at install time, not runtime-configurable

## Potential Issues

1. **No error recovery**: If `GUSMIDI_LoadPatch()` fails for a patch, the loop in `GUSMIDI_Init()` continues silently. Later `GUSMIDI_NoteOn()` may call `gf1_midi_note_on(NULL, ...)` with no warning.

2. **Unbounded preload**: All 256 patches are loaded regardless of which will be played. If GUS DRAM is fully saturated, `malloc()` may fail during `GUSMIDI_LoadPatch()`, but initialization doesn't abort—just silently skips.

3. **Config file rigidity**: `GUS_GetPatchMap()` expects exact format (`%d, %d, %d, %d, %d, %s`). A malformed line is skipped with no logging, making configuration errors silent.

4. **Interrupt safety incomplete**: Unload is interrupt-atomic, but load is not—if a note plays while a patch is being loaded, the GUS synthesis layer may access inconsistent data. Mitigation: patches are preloaded before any music plays.

5. **Static memory: Large, fixed arrays** (`patch_map[256][4]`, `program_name[256][9]`) consume ~3KB statically. In a 640K DOS heap, not critical, but inflexible if patch count needs to change.
