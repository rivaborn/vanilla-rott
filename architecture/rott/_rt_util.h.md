# rott/_rt_util.h

## File Purpose
A private utility header defining low-level VGA hardware constants, file paths for error/debug logging, and basic utility macros. Used for palette manipulation and debugging infrastructure in the Rise of the Triad engine.

## Core Responsibilities
- Define VGA hardware I/O port addresses for palette (PEL) read/write operations
- Provide file path constants for error logs and debug output
- Supply utility macros for sign determination and color weight calculations
- Define screen position constants for error message placement

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. File contains only preprocessor definitions and macros.

## Macros & Constants

| Name | Type | Purpose |
|------|------|---------|
| `PEL_WRITE_ADR`, `PEL_READ_ADR`, `PEL_DATA`, `PEL_MASK` | I/O port (0x3c6–0x3c9) | VGA hardware palette addressing for color lookup table read/write operations |
| `ERRORROW`, `ERRORCOL` | Screen coordinates | Display position for error messages (row 2, column 11) |
| `ERRORFILE`, `SOFTERRORFILE`, `DEBUGFILE`, `MAPDEBUGFILE` | File paths | Debug/error log outputs: ROTTERR.TXT, ERROR.*, ROTT.DBG, MAPINFO.TXT |
| `SGN(x)` | Macro | Returns sign of value: 1 if positive, 0 if zero, −1 if negative |
| `SLASHES`, `MAXCHARS` | Character constants | Backslash literal ('\\') and 8-character limit |
| `WeightR`, `WeightG`, `WeightB` | RGB color weights | Relative importance (3, 5, 2) for red/green/blue in color calculations |

## Control Flow Notes
This is a private definition header (indicated by `_rt_util_private` guard). The VGA I/O port constants suggest execution on DOS or DOSBox. Likely included by rendering, palette management, and error handling modules during initialization and runtime.

## External Dependencies
None—pure preprocessor definitions with no external symbols referenced.
