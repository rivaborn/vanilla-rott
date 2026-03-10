# audiolib/source/gusmidi2.c

## File Purpose

MIDI music driver for the Gravis Ultrasound audio card. Manages patch (instrument) loading/unloading, MIDI event dispatch, volume control, and hardware initialization. Enables the game engine to play General MIDI music via GUS synthesis hardware.

## Core Responsibilities

- Load and unload instrument patches from disk into GUS memory
- Parse ULTRAMID.INI configuration to map MIDI program numbers to patch files
- Dispatch MIDI events (note on/off, program change, control change, pitch bend) to GF1 driver
- Allocate and manage GUS DRAM across multiple hardware configurations (256K–1MB)
- Initialize/shutdown GUS hardware with reference counting
- Report errors through error code and human-readable message strings
- Control master volume (0–255 range)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `struct patch` | struct | Loaded instrument patch with layers (from newgf1.h) |
| `struct patchinfo` | struct | Patch file header and metadata (from newgf1.h) |
| `struct wave_struct` | struct | Individual waveform sample data (from newgf1.h) |
| `enum GUS_Errors` | enum | Error codes (GUS_Ok, GUS_OutOfMemory, etc.) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `new_patch` | `struct patch[256]` | static | Loaded instrument patches (melodic + drum) |
| `patch_waves` | `unsigned char FAR *[256]` | static | Pointers to waveform data for each patch |
| `patch_map` | `int[256][4]` | static | Maps MIDI program → patch index for 4 memory configs |
| `program_name` | `char[256][9]` | static | Patch file names for each program |
| `patch_flags` | `char[256]` | static | PATCH_LOADED flag per patch |
| `config_name` | `char[32]` | static | Config file name ("ULTRAMID.INI") |
| `ultradir` | `char[80]` | static | GUS base directory path from ULTRADIR env var |
| `hold_buffer` | `char*` | static | DMA buffer (2048 bytes) for patch file loading |
| `GUSMIDI_Volume` | `int` | static | Current master volume (0–255) |
| `GUS_TotalMemory` | `unsigned long` | static | Total GUS DRAM available |
| `GUS_MemConfig` | `int` | static | Memory config index (0–3) selected at init |
| `GUS_Installed` | `int` | static | Refcount for hardware initialization |
| `GUS_ErrorCode` | `int` | global | Current error code (from gusmidi.h) |
| `GUS_AuxError` | `int` | global | Auxiliary error code (errno, GF1 error) |

## Key Functions / Methods

### GUS_ErrorString
- **Signature:** `char *GUS_ErrorString(int ErrorNumber)`
- **Purpose:** Return human-readable error message for an error code.
- **Inputs:** Error code (or GUS_Error for current error).
- **Outputs/Return:** Pointer to static error message string.
- **Side effects:** None.
- **Calls:** `gf1_error_str()` for GF1-level errors.
- **Notes:** Maps GUS error enum to descriptive text. GUS_Error (-1) resolves the current `GUS_ErrorCode`.

### GUS_GetPatchMap
- **Signature:** `int GUS_GetPatchMap(char *name)`
- **Purpose:** Parse configuration file (ULTRAMID.INI) to map MIDI program numbers to patch files.
- **Inputs:** Config file name.
- **Outputs/Return:** GUS_Ok or error code (GUS_ULTRADIRNotSet, GUS_MissingConfig, GUS_FileError).
- **Side effects:** Initializes `patch_map[][]` with patch indices, populates `program_name[][]` with patch filenames.
- **Calls:** `getenv()`, `fopen()`, `fgets()`, `sscanf()`, `fseek()`, `fclose()`.
- **Notes:** Reads newline-delimited format: `patch_id, mem_cfg_0_idx, mem_cfg_1_idx, mem_cfg_2_idx, mem_cfg_3_idx, patch_filename`. Lines starting with '#' are comments. Initializes all entries to UNUSED_PATCH (-1) first.

### GUSMIDI_LoadPatch
- **Signature:** `int GUSMIDI_LoadPatch(int prognum)`
- **Purpose:** Load an instrument patch from disk into GUS DRAM.
- **Inputs:** MIDI program number (0–255).
- **Outputs/Return:** GUS_Ok or error code.
- **Side effects:** Allocates wave data, calls `gf1_load_patch()`, sets PATCH_LOADED flag, stores pointer in `patch_waves[]`.
- **Calls:** `gf1_get_patch_info()`, `malloc()`, `gf1_load_patch()`.
- **Notes:** Maps prognum through current `GUS_MemConfig`. Skips if patch already loaded, no filename, or prog == UNUSED_PATCH. Uses 2KB hold_buffer for DMA.

### GUSMIDI_UnloadPatch
- **Signature:** `int GUSMIDI_UnloadPatch(int prognum)`
- **Purpose:** Unload patch from GUS memory and free allocated resources.
- **Inputs:** MIDI program number.
- **Outputs/Return:** GUS_Ok.
- **Side effects:** Calls `gf1_unload_patch()`, `free(patch_waves[])`, disables/enables interrupts, zeros `nlayers` to silence active notes.
- **Calls:** `disable()`, `gf1_unload_patch()`, `free()`, `enable()`.
- **Notes:** Atomic with respect to interrupts. Zeros `nlayers` to ensure any playing notes are silenced immediately.

### GUSMIDI_ProgramChange
- **Signature:** `void GUSMIDI_ProgramChange(int channel, int prognum)`
- **Purpose:** Handle MIDI program change (instrument selection) on a channel.
- **Inputs:** MIDI channel, program number.
- **Outputs/Return:** None.
- **Side effects:** Calls `gf1_midi_change_program()`.
- **Calls:** `gf1_midi_change_program()`.
- **Notes:** If patch not loaded, passes NULL to GF1 driver.

### GUSMIDI_NoteOn
- **Signature:** `void GUSMIDI_NoteOn(int chan, int note, int velocity)`
- **Purpose:** Handle MIDI note-on event (start note playback).
- **Inputs:** MIDI channel, note number (0–127), velocity (0–127).
- **Outputs/Return:** None.
- **Side effects:** Calls `gf1_midi_note_on()`.
- **Calls:** `gf1_midi_note_on()`.
- **Notes:** Channel 9 is drums; uses `note + 128` for drum patch lookup. Other channels pass 0L (no patch).

### GUSMIDI_NoteOff
- **Signature:** `void GUSMIDI_NoteOff(int chan, int note, int velocity)`
- **Purpose:** Handle MIDI note-off event (stop note playback).
- **Inputs:** MIDI channel, note number, velocity.
- **Outputs/Return:** None.
- **Side effects:** Calls `gf1_midi_note_off()`.
- **Calls:** `gf1_midi_note_off()`.
- **Notes:** Velocity parameter unused (compile with pragma warn -par to suppress).

### GUSMIDI_ControlChange
- **Signature:** `void GUSMIDI_ControlChange(int channel, int number, int value)`
- **Purpose:** Handle MIDI control change (e.g., volume, pan, modulation).
- **Inputs:** MIDI channel, control number, value.
- **Outputs/Return:** None.
- **Side effects:** Calls `gf1_midi_parameter()`.
- **Calls:** `gf1_midi_parameter()`.

### GUSMIDI_PitchBend
- **Signature:** `void GUSMIDI_PitchBend(int channel, int lsb, int msb)`
- **Purpose:** Handle MIDI pitch bend event.
- **Inputs:** MIDI channel, LSB and MSB of pitch bend value (14-bit).
- **Outputs/Return:** None.
- **Side effects:** Calls `gf1_midi_pitch_bend()`.
- **Calls:** `gf1_midi_pitch_bend()`.

### GUSMIDI_SetVolume
- **Signature:** `void GUSMIDI_SetVolume(int volume)`
- **Purpose:** Set master MIDI volume.
- **Inputs:** Volume (0–255).
- **Outputs/Return:** None.
- **Side effects:** Clamps to [0, 255], stores in `GUSMIDI_Volume`, calls `gf1_midi_master_volume()` with range [0, 127] (right-shifted by 1).
- **Calls:** `max()`, `min()`, `gf1_midi_master_volume()`.

### GUSMIDI_GetVolume
- **Signature:** `int GUSMIDI_GetVolume(void)`
- **Purpose:** Retrieve current master volume.
- **Inputs:** None.
- **Outputs/Return:** Volume (0–255).
- **Side effects:** None.

### GUS_Init
- **Signature:** `int GUS_Init(void)`
- **Purpose:** Initialize GUS hardware (reference counted).
- **Inputs:** None.
- **Outputs/Return:** GUS_Ok or error code.
- **Side effects:** Increments `GUS_Installed` (refcount); on first init, calls `GetUltraCfg()` and `gf1_load_os()`.
- **Calls:** `GetUltraCfg()`, `gf1_load_os()`.
- **Notes:** Reference counted; subsequent calls before shutdown just increment refcount. Subsequent calls return immediately on refcount > 1.

### GUS_Shutdown
- **Signature:** `void GUS_Shutdown(void)`
- **Purpose:** Shutdown GUS hardware (reference counted).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Decrements `GUS_Installed`; when refcount reaches 0, calls `gf1_unload_os()`.
- **Calls:** `gf1_unload_os()`.
- **Notes:** Reference counted; true unload only on final shutdown.

### GUSMIDI_Init
- **Signature:** `int GUSMIDI_Init(void)`
- **Purpose:** Initialize MIDI subsystem—loads hardware, patches, and preloads all instruments.
- **Inputs:** None.
- **Outputs/Return:** GUS_Ok or error code.
- **Side effects:** Calls `GUS_Init()`, allocates `hold_buffer`, loads patch mapping, preloads all 256 patches into GUS DRAM.
- **Calls:** `GUS_Init()`, `gf1_mem_avail()`, `D32DosMemAlloc()`, `GUS_GetPatchMap()`, `GUSMIDI_LoadPatch()`.
- **Notes:** Selects memory config (0–3) based on available GUS DRAM: `GUS_MemConfig = (GUS_TotalMemory - 1) >> 18`. Loads all patches regardless of success (continues on error). Fails if hold_buffer allocation fails.

### GUSMIDI_Shutdown
- **Signature:** `void GUSMIDI_Shutdown(void)`
- **Purpose:** Shutdown MIDI subsystem.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `GUSMIDI_ReleasePatches()`, then `GUS_Shutdown()`.
- **Calls:** `GUSMIDI_ReleasePatches()`, `GUS_Shutdown()`.

### GUSMIDI_ReleasePatches
- **Signature:** `void GUSMIDI_ReleasePatches(void)`
- **Purpose:** Unload all loaded patches.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Unloads all 256 patches.
- **Calls:** `GUSMIDI_UnloadPatch()` (×256).

### D32DosMemAlloc
- **Signature:** `void *D32DosMemAlloc(unsigned size)`
- **Purpose:** Allocate conventional (real-mode) DOS memory, using DPMI if available.
- **Inputs:** Size in bytes.
- **Outputs/Return:** Pointer to allocated buffer, or NULL on failure.
- **Side effects:** Calls `int386()` (DPMI) on Watcom/FLAT or `malloc()` on other compilers.
- **Calls:** `int386()` or `malloc()`.
- **Notes:** Watcom 32-bit: uses DPMI int 0x31, function 0x0100 (allocate DOS memory). Others: fallback to malloc. Buffer must be < 64KB for real-mode DMA.

## Control Flow Notes

**Initialization chain:**
1. `GUSMIDI_Init()` → `GUS_Init()` → `gf1_load_os()` loads GUS driver.
2. Reads ULTRAMID.INI via `GUS_GetPatchMap()`.
3. Preloads all 256 patches via loop over `GUSMIDI_LoadPatch()`.

**Runtime:**
- MIDI events (note on/off, program change, etc.) are dispatched to GF1 driver via `GUSMIDI_*()` functions.
- Patch is selected from `patch_map[prognum][GUS_MemConfig]` based on available memory.

**Shutdown:**
- `GUSMIDI_Shutdown()` → `GUSMIDI_ReleasePatches()` (unload all) → `GUS_Shutdown()` (unload driver).

The module implements a reference-counted initialization pattern (GUS_Installed counter) to allow multiple subsystems to safely share hardware.

## External Dependencies

**Standard C:**
- `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<malloc.h>`, `<math.h>`, `<limits.h>`, `<io.h>`, `<fcntl.h>`

**DOS/platform-specific:**
- `<conio.h>`, `<dos.h>` — console I/O and DOS interrupts (for DPMI via `int386()`).

**Local headers:**
- `gusmidi.h` — public API and error codes.
- `newgf1.h` — GUS hardware abstraction and patch structures.

**External symbols (defined elsewhere):**
- `gf1_load_os()`, `gf1_unload_os()` — GUS driver load/unload.
- `gf1_get_patch_info()`, `gf1_load_patch()`, `gf1_unload_patch()` — patch management.
- `gf1_mem_avail()` — query available GUS memory.
- `gf1_midi_*()` — MIDI event dispatch (note on/off, program change, volume, pitch bend, etc.).
- `gf1_error_str()` — GF1 error message string.
- `GetUltraCfg()` — read hardware configuration from ULTRASND.INI.
