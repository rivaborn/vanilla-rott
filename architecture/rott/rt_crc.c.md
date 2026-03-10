# rott/rt_crc.c

## File Purpose
CRC-16 checksum calculation library for data integrity verification. Provides functions to compute Cyclic Redundancy Check values using a precomputed lookup table for efficient streaming and batch computation.

## Core Responsibilities
- Compute CRC-16 checksums for data buffers
- Support incremental (single-byte) CRC updates via lookup table
- Provide high-performance checksum calculation without dynamic allocation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| crc16tab | static array (unsigned short int[256]) | Precomputed CRC-16 lookup values for fast table-driven computation |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| crc16tab | unsigned short int[256] | static | Lookup table for CRC polynomial computation; 256 precomputed CRC values |

## Key Functions / Methods

### updatecrc
- **Signature:** `int updatecrc(int crc, int c)`
- **Purpose:** Incrementally update a CRC value with one additional byte.
- **Inputs:** `crc` (current CRC value), `c` (byte to incorporate)
- **Outputs/Return:** Updated CRC value (int)
- **Side effects:** None
- **Calls:** Direct table lookup only (crc16tab[])
- **Notes:** Core primitive using reverse-bit CRC-16 algorithm. XORs input byte with CRC low byte to index table for next CRC value.

### CalculateCRC
- **Signature:** `word CalculateCRC (byte *source, unsigned size)`
- **Purpose:** Compute CRC-16 checksum for an entire data buffer.
- **Inputs:** `source` (pointer to byte buffer), `size` (number of bytes)
- **Outputs/Return:** CRC-16 checksum as word
- **Side effects:** Advances source pointer; modifies local checksum and tmp variables
- **Calls:** Implements same logic as updatecrc inline
- **Notes:** Convenience wrapper that iterates through buffer calling updatecrc logic. Initializes checksum to 0, processes all bytes, casts result to word for return.

## Control Flow Notes
Utility library with no initialization or shutdown sequence. Functions are called on-demand by other engine subsystems for checksum validation:
- Likely invoked during file loading (verify data integrity)
- Used in network/streaming code (validate packet/message checksums)
- Called during memory/resource verification phases

## External Dependencies
- **Standard includes:** stdio.h, stdlib.h, string.h
- **Local includes:** rt_crc.h (function declarations), memcheck.h (debugging header, not functionally used)
- **Defined elsewhere:** `byte`, `word` types (likely in rt_def.h per rt_crc.h include chain)
