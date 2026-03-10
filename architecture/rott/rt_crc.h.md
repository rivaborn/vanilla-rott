# rott/rt_crc.h

## File Purpose
Header file declaring the CRC (Cyclic Redundancy Check) calculation interface. Provides function signatures for incremental CRC updates and bulk buffer CRC computation used for data integrity verification.

## Core Responsibilities
- Declare incremental CRC update function
- Declare block CRC calculation function for byte buffers
- Export CRC utility interface to other engine modules

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### updatecrc
- Signature: `int updatecrc(int, int)`
- Purpose: Incrementally update CRC state
- Inputs: Two int parameters (likely: current CRC value and next byte/value to incorporate)
- Outputs/Return: Updated int CRC state
- Side effects: None (pure computation)
- Calls: (Not inferable from this header)
- Notes: Parameter types/semantics not documented; implementation elsewhere

### CalculateCRC
- Signature: `word CalculateCRC(byte *source, unsigned size)`
- Purpose: Compute CRC checksum for a contiguous byte buffer
- Inputs: `source` (pointer to byte buffer), `size` (buffer length in bytes)
- Outputs/Return: `word` (16-bit CRC result)
- Side effects: None (read-only; pure computation)
- Calls: (Not inferable from this header; likely uses `updatecrc` internally)
- Notes: Convenience function wrapping incremental CRC loop; typical for data integrity checks in file I/O or network transmission

## Control Flow Notes
This is a header file; no control flow. Functions are entry points for other modules to compute or verify data checksums, likely during save/load operations or network validation.

## External Dependencies
- Includes: `rt_def.h` (provides `byte`, `word` type definitions)
- Implementation defined elsewhere (rt_crc.c presumed)
