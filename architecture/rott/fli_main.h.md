# rott/fli_main.h

## File Purpose
Header file for Flic animation file (.fli/.flc format) reading and playback. Defines the Flic structure, playback control interface, and error codes for managing Flic animations within the game engine.

## Core Responsibilities
- Define `Flic` structure for managing open Flic file state and playback
- Declare file I/O functions (open, close, block reading)
- Declare playback functions (single play, looping, frame advancement)
- Provide error reporting via error codes and string lookup
- Define utility macros for memory operations and array handling

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Flic | struct | State container for an open Flic file, including header, file handle, current frame, display offset, memory buffer pointer, and playback mode |

## Global / File-Static State
None.

## Key Functions / Methods

### flic_open
- Signature: `ErrCode flic_open(Flic *flic, char *name, MemPtr buf, Boolean usefile)`
- Purpose: Open and initialize a Flic file, read header, verify format
- Inputs: `flic` (pointer to Flic structure), `name` (filename), `buf` (memory buffer), `usefile` (use file or memory)
- Outputs/Return: `ErrCode` (status code)
- Side effects: Allocates/initializes Flic structure, opens file handle, reads header

### flic_close
- Signature: `void flic_close(Flic *flic)`
- Purpose: Close Flic file and clean up associated resources
- Inputs: `flic` (pointer to Flic structure)
- Side effects: Closes file handle, deallocates/clears Flic structure

### flic_play_once
- Signature: `ErrCode flic_play_once(Flic *flic, Machine *machine)`
- Purpose: Play Flic animation through once to completion
- Inputs: `flic` (Flic file), `machine` (render/display target)
- Outputs/Return: `ErrCode`
- Side effects: Updates display, advances animation frames

### flic_play_loop
- Signature: `ErrCode flic_play_loop(Flic *flic, Machine *machine)`
- Purpose: Play Flic animation in a loop until interrupted
- Inputs: `flic`, `machine`
- Outputs/Return: `ErrCode`
- Side effects: Updates display repeatedly, responds to input events

### flic_next_frame
- Signature: `ErrCode flic_next_frame(Flic *flic, Screen *screen)`
- Purpose: Advance to next frame and render to screen
- Inputs: `flic`, `screen`
- Outputs/Return: `ErrCode`

### SetupFlicAccess
- Signature: `ErrCode SetupFlicAccess(Flic *flic)`
- Purpose: Prepare internal structures for subsequent Flic data access

### CopyNextFlicBlock
- Signature: `ErrCode CopyNextFlicBlock(Flic *flic, MemPtr buf, Ulong size)`
- Purpose: Copy next chunk of Flic data to destination buffer
- Inputs: `flic`, `buf` (destination), `size` (bytes to read)

### SetFlicOffset
- Signature: `void SetFlicOffset(Flic *flic, Ulong offset)`
- Purpose: Seek to specified offset within Flic file

### flic_err_string
- Signature: `char *flic_err_string(ErrCode err)`
- Purpose: Retrieve human-readable error message for error code

## Control Flow Notes
Flic module integrates into initialization/playback phase. Typical flow: `flic_open()` → `SetupFlicAccess()` → repeated `flic_next_frame()` calls or single `flic_play_once()`/`flic_play_loop()` → `flic_close()`. Used for intro/outro animations and cinematic sequences.

## External Dependencies
- **Defined elsewhere**: `FlicHead` (struct), `Machine` (struct), `Screen` (struct), `MemPtr` (typedef), `Boolean` (typedef), `ErrCode` (typedef)
- **Standard C**: `memset()` (via `ClearMem` macro)
- **Attribution**: Flic reader based on Jim Kent code (1992), adapted for Apogee engine
