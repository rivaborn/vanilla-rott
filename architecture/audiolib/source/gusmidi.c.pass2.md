# audiolib/source/gusmidi.c — Enhanced Analysis

## Architectural Role

GUSMIDI is a **hardware driver adapter** that bridges the Gravis Ultrasound (GUS) synthesizer hardware with MIDI message protocols. It sits between a hypothesized higher-level MIDI abstraction layer (likely in `al_midi.c`) and the low-level GF1 driver (`newgf1.h`), mediating all instrument loading, state management, and real-time MIDI message dispatch. The file is one of several parallel hardware backends (ADLIBFX, AWE32, BLASTER in the cross-reference index), indicating the audio library supports multiple sound cards through a pluggable driver architecture.

## Key Cross-References

### Incoming (who depends on this file)
- **Upper-level MIDI dispatcher** (likely `al_midi.c` based on cross-reference function names `AL_ProgramChange`, `AL_NoteOn`, etc.): Calls `GUSMIDI_ProgramChange()`, `GUSMIDI_NoteOn()`, `GUSMIDI_NoteOff()`, `GUSMIDI_ControlChange()`, `GUSMIDI_PitchBend()` when the GUS backend is active
- **Main game engine initialization**: Calls `GUSMIDI_Init()` during startup; calls `GUSMIDI_Shutdown()` during cleanup
- **Volume control subsystem**: Calls `GUSMIDI_SetVolume()` and `GUSMIDI_GetVolume()` from game menus or settings
- **Game lifecycle manager**: Calls `GUSMIDI_ReleasePatches()` when switching scenes or shutting down audio

### Outgoing (what this file depends on)
- **GF1 driver** (`newgf1.h`): Calls `gf1_get_patch_info()`, `gf1_load_patch()`, `gf1_unload_patch()`, `gf1_midi_*()` functions (note_on, note_off, change_program, parameter, pitch_bend, synth_volume)
- **GUS hardware management** (`newgf1.h`): Reads/writes `GUS_MemConfig`, `GUS_TotalMemory`, `GUS_ErrorCode`, `GUS_AuxError`; uses `GUS_HoldBuffer` (shared DMA staging area)
- **GUS initialization** (`newgf1.h`): Calls `GUS_Init()` and `GUS_Shutdown()` for hardware power-on/off
- **Memory hooks** (`usrhooks.h`): Calls `USRHOOKS_GetMem()` and `USRHOOKS_FreeMem()` for system RAM allocation (waveform buffers)
- **Interrupt control** (`interrup.h`): Calls `DisableInterrupts()` / `RestoreInterrupts()` during patch unload (synchronization with ISRs)
- **Standard I/O**: `fopen()`, `fgets()`, `sscanf()` for config file parsing; `getenv()` for environment variables

## Design Patterns & Rationale

**Hardware Adapter**: GUSMIDI wraps the raw GF1 driver, providing a cleaner MIDI vocabulary (`NoteOn`, `NoteOff`, `ProgramChange`) that hides GUS-specific details (patch indirection, memory config selection, DMA buffers).

**Preload-All Resource Strategy**: All 256 patches (melodic + percussion) are loaded into GUS DRAM at `GUSMIDI_Init()` time. This trades startup latency for guaranteed real-time performance—no disk I/O during gameplay, critical for DOS-era responsiveness. The design gracefully degrades: if individual patches fail to load, the system continues with partial instrumentation (passing NULL to the driver for missing patches).

**Configuration-Driven Mapping**: The `PatchMap[program][memconfig]` indirection allows a single game binary to support different GUS hardware variants (256KB, 512KB, 1MB, 2MB configs) by loading different INI files. This was a pragmatic solution before platform-agnostic game engines.

**Interrupt-Safe Patch Unload**: The `DisableInterrupts()` wrapper around `gf1_unload_patch()` and the explicit `nlayers=0` guard prevent concurrent access (ISR vs. main thread). Setting `nlayers=0` before freeing ensures the GF1 driver won't dereference freed `PatchWaves` buffers if a note is still being played.

## Data Flow Through This File

**Initialization Phase**:
```
GUSMIDI_Init()
  → GUS_Init() (power on hardware)
  → GUS_GetPatchMap("ULTRAMID.INI")
      → parse config file, populate PatchMap[0..255][0..3], ProgramName[0..255]
  → GUSMIDI_LoadPatch() ×256
      → allocate waveform buffer in system RAM
      → read .PAT file header via gf1_get_patch_info()
      → transfer to GUS DRAM via gf1_load_patch() (using GUS_HoldBuffer DMA)
      → record loaded state, free or reuse waveform buffer
  → GUSMIDI_SetVolume(default_volume)
```

**Runtime MIDI Dispatch**:
```
Higher-level dispatcher
  → GUSMIDI_ProgramChange(channel, program)
      → lookup PatchMap[program][GUS_MemConfig] to get hardware patch
      → call gf1_midi_change_program() to route channel to patch
  → GUSMIDI_NoteOn(channel, note, velocity)
      → (special case: channel 9 = percussion, note offset to 128+ range)
      → call gf1_midi_note_on() (synthesizer plays note from current patch)
  → GUSMIDI_NoteOff(channel, note, velocity)
  → GUSMIDI_ControlChange(channel, controller, value)
      → gf1_midi_parameter() (modulation, volume, etc.)
  → GUSMIDI_PitchBend(channel, lsb, msb)
```

**Shutdown Phase**:
```
GUSMIDI_Shutdown()
  → GUSMIDI_ReleasePatches()
      → GUSMIDI_UnloadPatch() ×256
          → DisableInterrupts()
          → gf1_unload_patch()
          → USRHOOKS_FreeMem(PatchWaves)
          → RestoreInterrupts()
  → GUS_Shutdown() (power down hardware)
```

## Learning Notes

**For engine programmers**:
- Shows how to adapt low-level hardware drivers to standard MIDI abstractions
- Demonstrates preload resource strategies—valuable for understanding performance tradeoffs in real-time systems
- Illustrates interrupt safety in ISR-aware code (modern multithreading makes this less critical, but the pattern persists)

**DOS-era idioms**:
- Explicit `struct` packing pragmas (`zp1`)—modern aligned allocators hide this
- Environment variables (`ULTRADIR`) as configuration—replaced by config files/registries in later eras
- DMA buffers for hardware transfers—now handled by kernels/drivers
- No exceptions or assertions in error paths—static error codes returned
- Interrupt disabling for synchronization—replaced by mutexes/atomics in modern OS

**MIDI vocabulary**:
- Program change = select instrument
- Note on/off = trigger/stop voices
- Control change = continuous parameters (CC7=volume, CC10=pan, etc.)
- Pitch bend = 14-bit pitch offset
- Percussion (channel 9) uses note number as drum kit index

**Architectural patterns**:
- **Pluggable backends**: Parallel drivers (GUSMIDI, AWE32, ADLIBFX) suggest runtime selection based on detected hardware
- **Configuration-driven behavior**: Patch maps are loaded from INI, not hardcoded—supports multiple hardware revisions and regional tuning
- **Graceful degradation**: Missing patches don't crash; they produce silence, allowing the game to continue

## Potential Issues

**Config parsing vulnerability**: `sscanf()` into `ProgramName[index]` with format `%s` can overflow if a .INI entry exceeds 8 characters. No bounds checking. This is mitigated by assuming trusted input (internal toolchain), but modern code would use `%8s` or safer parsers.

**Silent patch load failures**: If `GUSMIDI_LoadPatch()` fails for individual patches, the loop continues without logging. The game will have missing instruments but won't abort. Modern engines might fail-fast or fall back to a default synth.

**No runtime patch reload**: GUS DRAM corruption or hardware reset during gameplay cannot be detected or recovered. Acceptable for unattended arcade machines, risky for home use.

**Percussion note offset**: Channel 9 note numbers are offset by +128 to use `PatchMap[note+128][]` for drum kit lookup. This hardcoded offset is clear but inflexible (e.g., cannot support multiple drum kits per channel without refactoring).
