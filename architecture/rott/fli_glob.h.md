# rott/fli_glob.h

## File Purpose
A header file declaring the interface for playing FLI/FLIC animation files. FLI is a video format commonly used in 1990s games for cutscenes and cinematics.

## Core Responsibilities
- Declares the `PlayFlic` function for animation playback
- Provides flexible media playback interface supporting both file-based and memory-based animation sources

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### PlayFlic
- **Signature:** `void PlayFlic ( char * name, unsigned char * buffer, int usefile, int loop)`
- **Purpose:** Plays a FLI/FLIC animation either from a file or from a memory buffer.
- **Inputs:**
  - `name`: Pointer to string; likely filename or animation identifier
  - `buffer`: Pointer to unsigned char array; animation data in memory
  - `usefile`: Integer flag (likely boolean: non-zero = use file, zero = use buffer)
  - `loop`: Integer flag (likely boolean: non-zero = loop animation, zero = play once)
- **Outputs/Return:** Void; no return value.
- **Side effects:** Renders animation frames; likely modifies display state during playback.
- **Calls:** Not inferable from this file (implementation elsewhere).
- **Notes:** Dual-mode design allows loading from disk or pre-buffered memory. Implementation details (blocking vs. asynchronous, frame rate handling) not visible in header.

## Control Flow Notes
Typical usage context: cutscene playback, intro cinematics, or menu animations. Whether this function blocks until animation completes or returns immediately is not inferable from this header.

## External Dependencies
- Standard C library (function signature only; details in implementation)
- Implementation expected in `fli_glob.c`
- No includes visible in this header file
