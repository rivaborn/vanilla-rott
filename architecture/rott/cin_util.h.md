# rott/cin_util.h

## File Purpose
Header file providing the public interface for cinematic palette operations. Declares functions to read and write the game's cinematic palette state.

## Core Responsibilities
- Define public API for cinematic palette management
- Expose palette getter/setter functions to the cinematic subsystem

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### CinematicGetPalette
- Signature: `void CinematicGetPalette (byte *pal)`
- Purpose: Retrieve the current cinematic palette data
- Inputs: Pointer to a byte buffer where palette will be stored
- Outputs/Return: None (writes to provided buffer)
- Side effects: Reads cinematic palette state (possibly from memory or video hardware)
- Calls: Not inferable from this file
- Notes: Caller must allocate sufficient buffer; expected size not documented

### CinematicSetPalette
- Signature: `void CinematicSetPalette (byte *pal)`
- Purpose: Apply a palette to the cinematic display system
- Inputs: Pointer to byte buffer containing palette data
- Outputs/Return: None
- Side effects: Modifies cinematic palette state; likely updates video display hardware
- Calls: Not inferable from this file
- Notes: Caller must provide valid palette data; size not documented

## Control Flow Notes
Not inferable from this file. These are utility functions likely called during cinematic setup/teardown or palette transitions.

## External Dependencies
- `byte` type (defined elsewhere, likely in a common header like `_types.h`)
