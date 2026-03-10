# audiolib/source/pitch.h

## File Purpose
Public header for the PITCH module, which handles pitch scaling calculations and memory management for audio pitch operations. This is part of the original Apogee Software audio library from 1994.

## Core Responsibilities
- Define pitch operation error codes
- Declare pitch scaling calculation interface
- Manage memory lifecycle (lock/unlock) for pitch data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PITCH_ERRORS | enum | Error codes for pitch operations (Warning, Error, Ok) |

## Global / File-Static State
None.

## Key Functions / Methods

### PITCH_GetScale
- Signature: `unsigned long PITCH_GetScale(int pitchoffset)`
- Purpose: Calculate and return the pitch scale value for a given pitch offset
- Inputs: `pitchoffset` — integer offset from a base pitch value
- Outputs/Return: `unsigned long` — computed pitch scale factor
- Side effects: Not inferable from header
- Calls: Not inferable from header
- Notes: Return type is `unsigned long`; implementation details in PITCH.C

### PITCH_LockMemory
- Signature: `int PITCH_LockMemory(void)`
- Purpose: Lock pitch-related memory to prevent swapping/relocation (likely for real-time audio constraints)
- Inputs: None
- Outputs/Return: `int` — status code (0 likely indicates success)
- Side effects: Allocates or protects memory; affects resource state
- Calls: Not inferable from header

### PITCH_UnlockMemory
- Signature: `void PITCH_UnlockMemory(void)`
- Purpose: Unlock previously locked pitch memory
- Inputs: None
- Outputs/Return: None
- Side effects: Releases memory lock; affects resource state
- Calls: Not inferable from header

## Control Flow Notes
Lock/unlock pair suggests memory management around pitch operations. Likely used during audio initialization (lock) and shutdown (unlock). `PITCH_GetScale` appears to be a utility for real-time pitch calculations.

## External Dependencies
- No explicit includes shown; implementation in PITCH.C
- Note: `PITCH_Init()` is declared but commented out, suggesting initialization may be handled elsewhere or was removed
