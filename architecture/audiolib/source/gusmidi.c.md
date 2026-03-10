# audiolib/source/gusmidi.c

## File Purpose
Implements MIDI music playback for the Gravis Ultrasound sound card. Handles loading instrument patches from disk (.pat files), managing MIDI message routing (program changes, note on/off, pitch bend, control changes), and providing initialization/shutdown routines for the GUS hardware.

## Core Responsibilities
- Load and unload instrument patches from disk into GUS DRAM, with interrupt-safe memory management
- Parse MIDI configuration files (ULTRAMID.INI) to map MIDI program numbers to hardware patch indices
- Route MIDI messages (program change, note on/off, control change, pitch bend) to the GF1 driver
- Manage master volume control for MIDI synthesis
- Maintain patch metadata: filenames, load status, waveform memory pointers across 256 melodic and percussion instruments
- Initialize and shutdown GUS hardware with configuration validation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `patch` | struct (external) | Loaded instrument with layers and waveform data |
| `patchinfo` | struct (external) | File header and metadata read from .pat file |
| `wave_struct` | struct (external) | Single waveform descriptor within a patch (sample rate, loop points, envelope) |
| `gf1_dma_buff` | struct (external) | DMA buffer descriptor for transferring patch data to GUS memory |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Patch` | `struct patch[256]` | static | Loaded patches (indices 0–127 melodic, 128–255 percussion) |
| `PatchWaves` | `unsigned char*[256]` | static | Pointers to wave sample memory for each patch |
| `PatchMap` | `int[256][4]` | static | Maps MIDI programs to patch indices across 4 memory configs |
| `ProgramName` | `char[256][9]` | static | Patch filenames (.pat names, max 8 chars + null) |
| `PatchLoaded` | `char[256]` | static | Boolean flags: patch currently in GUS DRAM |
| `ConfigFileName` | `char*` | static | Hardcoded: "ULTRAMID.INI" |
| `ConfigDirectory` | `char[80]` | static | Directory path for config (from ULTRADIR env var) |
| `InstrumentDirectory` | `char[80]` | static | Directory path for .pat files (from ULTRADIR\midi\) |
| `GUSMIDI_Volume` | `int` | static | Master volume (2–255 range; 0–127 sent to GUS) |
| `GUSMIDI_Installed` | `int` | global | Boolean: initialization complete and active |
| `GUS_HoldBuffer` | `struct gf1_dma_buff` | extern | DMA staging buffer for patch loads |
| `GUS_TotalMemory`, `GUS_MemConfig` | extern | extern | GUS memory size and current config index (0–3) |
| `GUS_ErrorCode`, `GUS_AuxError` | `int` | extern | Error reporting from GUS driver and config loading |

## Key Functions / Methods

### GUS_GetPatchMap
- **Signature**: `int GUS_GetPatchMap(char *name)`
- **Purpose**: Parse MIDI configuration file and populate patch map from disk
- **Inputs**: `name` — filename (absolute or relative to ULTRADIR\midi\)
- **Outputs/Return**: `GUS_Ok` on success; `GUS_Error` if file not found or ULTRADIR unset
- **Side effects**: Sets `InstrumentDirectory`, `ConfigDirectory`, populates `PatchMap[][]` and `ProgramName[][]`; sets `GUS_ErrorCode` on error
- **Calls**: `getenv()`, `fopen()`, `fgets()`, `fclose()`, `sscanf()`, string functions
- **Notes**: Config format: `index, map0, map1, map2, map3, name` (# for comments). Initializes all maps to `UNUSED_PATCH` (−1) first. Supports up to 4 memory configurations.

### GUSMIDI_LoadPatch
- **Signature**: `int GUSMIDI_LoadPatch(int prognum)`
- **Purpose**: Load a single instrument patch from disk into GUS DRAM via DMA
- **Inputs**: `prognum` — MIDI program number (0–255)
- **Outputs/Return**: `GUS_Ok` on success; `GUS_Error` on memory exhaustion or GF1 failure
- **Side effects**: Allocates waveform buffer in system RAM, transfers to GUS, updates `PatchLoaded[prog]` and `PatchWaves[prog]`; sets error codes on failure
- **Calls**: `USRHOOKS_GetMem()`, `gf1_get_patch_info()`, `gf1_load_patch()`, `USRHOOKS_FreeMem()` (on error)
- **Notes**: Skips if already loaded or if mapped patch is `UNUSED_PATCH`. Allocates buffer for waveforms: `patchi.header.wave_forms * sizeof(struct wave_struct)`. Uses 8-bit loading.

### GUSMIDI_UnloadPatch
- **Signature**: `int GUSMIDI_UnloadPatch(int prognum)`
- **Purpose**: Remove patch from GUS DRAM and free waveform memory
- **Inputs**: `prognum` — MIDI program number
- **Outputs/Return**: Always `GUS_Ok`
- **Side effects**: Disables interrupts during unload; calls `gf1_unload_patch()`, frees `PatchWaves` RAM, sets `nlayers=0` (silence), clears `PatchLoaded` flag
- **Calls**: `DisableInterrupts()`, `RestoreInterrupts()`, `gf1_unload_patch()`, `USRHOOKS_FreeMem()`
- **Notes**: Interrupt protection prevents concurrent access. Setting `nlayers=0` stops the driver from referencing freed patch.

### GUSMIDI_ProgramChange
- **Signature**: `void GUSMIDI_ProgramChange(int channel, int prognum)`
- **Purpose**: Select instrument on a MIDI channel
- **Inputs**: `channel` — MIDI channel (0–15); `prognum` — program (0–127)
- **Outputs/Return**: None
- **Side effects**: Synth program assignment on GUS
- **Calls**: `gf1_midi_change_program()`
- **Notes**: Passes NULL if patch unloaded (silences channel).

### GUSMIDI_NoteOn
- **Signature**: `void GUSMIDI_NoteOn(int chan, int note, int velocity)`
- **Purpose**: Start a note on a MIDI channel
- **Inputs**: `chan` — channel (0–15, 9=drums); `note` — MIDI note (0–127); `velocity` — volume (0–127)
- **Outputs/Return**: None
- **Side effects**: Triggers GUS voice for playback
- **Calls**: `gf1_midi_note_on()`
- **Notes**: Percussion channel (9) uses `PatchMap[note+128]` for drum kit lookup.

### GUSMIDI_NoteOff
- **Signature**: `void GUSMIDI_NoteOff(int chan, int note, int velocity)`
- **Purpose**: Stop a note
- **Inputs**: `chan`, `note`, `velocity` (velocity unused)
- **Outputs/Return**: None
- **Calls**: `gf1_midi_note_off()`

### GUSMIDI_ControlChange
- **Signature**: `void GUSMIDI_ControlChange(int channel, int number, int value)`
- **Purpose**: Forward MIDI continuous controller message to GUS
- **Inputs**: `channel`, `number` (controller type), `value`
- **Outputs/Return**: None
- **Calls**: `gf1_midi_parameter()`

### GUSMIDI_PitchBend
- **Signature**: `void GUSMIDI_PitchBend(int channel, int lsb, int msb)`
- **Purpose**: Set pitch bend on a channel
- **Inputs**: `channel`, `lsb`, `msb` (14-bit pitch value)
- **Outputs/Return**: None
- **Calls**: `gf1_midi_pitch_bend()`

### GUSMIDI_ReleasePatches
- **Signature**: `void GUSMIDI_ReleasePatches(void)`
- **Purpose**: Unload all 256 patches (melodic and percussion)
- **Inputs**: None
- **Calls**: `GUSMIDI_UnloadPatch()` (256 iterations)

### GUSMIDI_SetVolume / GUSMIDI_GetVolume
- **Signature**: `void GUSMIDI_SetVolume(int volume)`; `int GUSMIDI_GetVolume(void)`
- **Purpose**: Set and get master MIDI playback volume
- **Side effects (SetVolume)**: Clamps input to [2, 255]; divides by 2 for GUS driver (0–127 range); stores in `GUSMIDI_Volume`
- **Calls (SetVolume)**: `gf1_midi_synth_volume()`
- **Notes**: Minimum 2 (not 0) due to GUS tremolo artifact.

### GUSMIDI_Init
- **Signature**: `int GUSMIDI_Init(void)`
- **Purpose**: Initialize Gravis Ultrasound for MIDI playback
- **Inputs**: None
- **Outputs/Return**: `GUS_Ok` on success; `GUS_Error` on hardware or config failure
- **Side effects**: Calls `GUS_Init()`, validates `GUS_MemConfig` (clamps to [0, MAX_MEM_CONFIG]), resets patch arrays, loads config via `GUS_GetPatchMap()`, loads all 256 patches, sets `GUSMIDI_Installed=TRUE`
- **Calls**: `GUS_Init()`, `GUS_GetPatchMap()`, `GUSMIDI_SetVolume()`, `gf1_mem_avail()` (diagnostics), `GUSMIDI_LoadPatch()` (×256), `GUSMIDI_Shutdown()` (on error)
- **Notes**: Re-initializes safely (shutdowns first). Continues loading patches on individual failures (errors ignored, loop completes).

### GUSMIDI_Shutdown
- **Signature**: `void GUSMIDI_Shutdown(void)`
- **Purpose**: Shut down GUS MIDI system
- **Inputs**: None
- **Side effects**: Unloads all patches, calls `GUS_Shutdown()`, sets `GUSMIDI_Installed=FALSE`
- **Calls**: `GUSMIDI_ReleasePatches()`, `GUS_Shutdown()`
- **Notes**: Safe to call multiple times.

## Control Flow Notes
**Init phase**: `GUSMIDI_Init()` → `GUS_Init()` (hardware) → `GUS_GetPatchMap()` (parse config) → `GUSMIDI_LoadPatch()` ×256 (preload all patches into GUS)  
**Runtime**: MIDI message handlers (`NoteOn`, `NoteOff`, `ControlChange`, `PitchBend`, `ProgramChange`) forward to GF1 driver, which manages live voice synthesis.  
**Shutdown**: `GUSMIDI_Shutdown()` → `GUSMIDI_ReleasePatches()` → unload all → `GUS_Shutdown()`

Frame integration and message dispatch loop not inferable from this file.

## External Dependencies
**Standard headers**: `<conio.h>`, `<dos.h>`, `<stdio.h>`, `<io.h>`, `<fcntl.h>`, `<string.h>`, `<stdlib.h>` (DOS/Turbo C era)  
**Local headers**:
- `"usrhooks.h"` — Memory allocation hooks (`USRHOOKS_GetMem`, `USRHOOKS_FreeMem`)
- `"interrup.h"` — Interrupt control macros (`DisableInterrupts`, `RestoreInterrupts`)
- `"newgf1.h"` — GUS driver API (`gf1_*` functions, structures)
- `"gusmidi.h"` — Module public interface and error codes

**External symbols (defined elsewhere)**:
- `GUS_HoldBuffer`, `GUS_TotalMemory`, `GUS_MemConfig`, `GUS_ErrorCode`, `GUS_AuxError` — GUS driver state
- `GUS_Init()`, `GUS_Shutdown()` — Hardware initialization
- All `gf1_*()` functions — Gravis Ultrasound driver API (patch I/O, MIDI synthesis, DMA)
