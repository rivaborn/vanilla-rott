# audiolib/source/gusmidi.h

## File Purpose
Header file defining the public interface for Gravis UltraSound (GUS) MIDI control. Declares error codes, MIDI note/control functions, patch management, and initialization routines for sound card hardware abstraction.

## Core Responsibilities
- Define GUS error/status codes enum for return values
- Declare MIDI event functions (note on/off, pitch bend, control change, program change)
- Declare patch loading/unloading and mapping functions
- Declare volume control functions
- Declare GUS hardware initialization and shutdown
- Provide error string conversion utility

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GUS_Errors` | enum | Status/error codes returned by GUS functions (range: -2 to ~8) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `GUS_HoldBuffer` | `struct gf1_dma_buff` | extern | DMA buffer for GUS hardware data transfers |

## Key Functions / Methods

### GUS_Init
- **Signature:** `int GUS_Init(void)`
- **Purpose:** Initialize GUS hardware device
- **Outputs/Return:** `GUS_Errors` status code
- **Side effects:** Hardware initialization, state setup

### GUSMIDI_Init
- **Signature:** `int GUSMIDI_Init(void)`
- **Purpose:** Initialize MIDI subsystem on GUS
- **Outputs/Return:** `GUS_Errors` status code

### GUSMIDI_NoteOn / GUSMIDI_NoteOff
- **Signature:** `void GUSMIDI_NoteOn(int chan, int note, int velocity)` / `void GUSMIDI_NoteOff(int chan, int note, int velocity)`
- **Purpose:** Trigger/release a note on the specified channel
- **Inputs:** MIDI channel (0-15), note number (0-127), velocity (0-127)

### GUSMIDI_ProgramChange
- **Signature:** `void GUSMIDI_ProgramChange(int channel, int prog)`
- **Purpose:** Select instrument patch for a MIDI channel
- **Inputs:** Channel, program number

### GUSMIDI_PitchBend / GUSMIDI_ControlChange
- **Signature:** `void GUSMIDI_PitchBend(int channel, int lsb, int msb)` / `void GUSMIDI_ControlChange(int channel, int number, int value)`
- **Purpose:** Send pitch bend or continuous controller messages

### GUSMIDI_LoadPatch / GUSMIDI_UnloadPatch
- **Signature:** `int GUSMIDI_LoadPatch(int prog)` / `int GUSMIDI_UnloadPatch(int prog)`
- **Purpose:** Load/unload instrument patch into GUS memory
- **Outputs/Return:** `GUS_Errors` status code

### GUS_ErrorString
- **Signature:** `char *GUS_ErrorString(int ErrorNumber)`
- **Purpose:** Convert error code to human-readable string
- **Inputs:** Error code from `GUS_Errors` enum
- **Outputs/Return:** Pointer to error message string

## Control Flow Notes
**Initialization phase:** `GUS_Init()` → `GUSMIDI_Init()` → `GUSMIDI_LoadPatch()` (for each instrument)  
**Runtime (per frame/MIDI event):** `GUSMIDI_NoteOn/Off()`, `GUSMIDI_PitchBend()`, `GUSMIDI_ControlChange()`, `GUSMIDI_SetVolume()`  
**Shutdown:** `GUSMIDI_ReleasePatches()` → `GUSMIDI_Shutdown()` → `GUS_Shutdown()`

## External Dependencies
- `struct gf1_dma_buff` (GF1 refers to GUS chipset; struct defined elsewhere)
- Watcom C compiler pragmas (`#pragma aux`) for low-level hardware calling conventions
- DOS memory allocation patterns (`D32DosMemAlloc`)
- Apogee Software copyright (1994-1995)

**Notes:** This is legacy DOS-era code targeting the Gravis UltraSound card via Watcom C. The pragmas and DOS memory references indicate real-mode or protected-mode DOS compilation.
