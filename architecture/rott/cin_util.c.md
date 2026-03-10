# rott/cin_util.c

## File Purpose
Provides utility functions for reading and writing VGA color palettes during cinematic playback. These functions directly access VGA hardware I/O ports to synchronize palette state between the engine and display hardware.

## Core Responsibilities
- Read 8-bit VGA color palette from hardware ports into application memory
- Write 8-bit color palette from application memory to hardware ports
- Handle bit-shift scaling between VGA's 6-bit internal representation and the 8-bit storage format
- Support cinematic sequences that require precise color control

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### CinematicGetPalette
- Signature: `void CinematicGetPalette(byte *pal)`
- Purpose: Read the current VGA color palette from hardware and store in caller's buffer
- Inputs: `pal` – pointer to 768-byte output buffer (256 colors × 3 channels)
- Outputs/Return: none (result written to `*pal`)
- Side effects: 
  - I/O port read from VGA palette hardware (0x3c7, 0x3c9)
  - Modifies caller's buffer
- Calls: `outp()`, `inp()` (conio.h)
- Notes: 
  - Each hardware color component is 6-bit; shifts left by 2 to scale to 8-bit
  - Loop iterates exactly 768 times (256 colors × RGB)
  - Initializes PEL_READ_ADR to 0 before reading, indicating palette index 0

### CinematicSetPalette
- Signature: `void CinematicSetPalette(byte *pal)`
- Purpose: Write a color palette from application memory to VGA hardware
- Inputs: `pal` – pointer to 768-byte palette buffer (256 colors × 3 channels)
- Outputs/Return: none (result written to VGA hardware)
- Side effects: 
  - I/O port write to VGA palette hardware (0x3c8, 0x3c9)
  - Changes display appearance by altering all 256 palette entries
- Calls: `outp()` (conio.h)
- Notes: 
  - Each application buffer entry is 8-bit; shifts right by 2 to scale to VGA's 6-bit format
  - Loop iterates exactly 768 times
  - Initializes PEL_WRITE_ADR to 0 before writing, indicating palette index 0

## Control Flow Notes
These functions are invoked during cinematic (FMV) playback to manage palette transitions and color effects. They interact with the VGA hardware subsystem initialized elsewhere (via `modexlib.h` Mode X functions). Called on the frame-update path when cinematic frames require palette adjustments.

## External Dependencies
- `conio.h` – provides `outp()` (write to I/O port) and `inp()` (read from I/O port)
- `modexlib.h` – defines VGA hardware port constants (`PEL_READ_ADR`, `PEL_WRITE_ADR`, `PEL_DATA`)
- `cin_glob.h` – declares cinematic subsystem declarations and includes
- `memcheck.h` – memory debug/check facilities (included but unused in this file)
