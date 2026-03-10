# rott/_w_wad.h

## File Purpose
Private header defining the WAD file format structures and constants. WAD files are the game's resource containers (lumps) used for maps, sprites, textures, and other assets. Provides structures for parsing WAD headers and lump directory entries.

## Core Responsibilities
- Define WAD file header structure (`wadinfo_t`) for identifying and locating lump directories
- Define in-memory lump metadata structure (`lumpinfo_t`) for tracking loaded resources
- Define WAD file lump directory entry structure (`filelump_t`) for reading from disk
- Provide compile-time WAD checksum validation constants (varies by build config: Shareware/Deluxe/Low-cost)
- Specify lump check period for periodic validation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `lumpinfo_t` | struct | In-memory representation of a loaded lump: name (8 chars), file handle, position, size |
| `wadinfo_t` | struct | WAD file header: 4-byte identification (IWAD), lump count, offset to lump directory table |
| `filelump_t` | struct | WAD file lump directory entry: file position, size, 8-char name |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `CHECKPERIOD` | #define | file-static | Frequency constant (20) for periodic lump validation checks |
| `WADCHECKSUM` | #define | file-static | Build-variant integrity checksum; differs for Shareware (45677), Deluxe (54748), Low-cost (12185), and full versions (24222) |

## Key Functions / Methods
None. This is a pure data structure definition header.

## Control Flow Notes
This header is used during game initialization/loading phase. A WAD loader (defined elsewhere) would:
1. Read `wadinfo_t` from file to locate the lump directory
2. Read `filelump_t` entries to populate `lumpinfo_t` structures
3. Use checksums for WAD integrity validation
4. Periodically check lumps every `CHECKPERIOD` operations

## External Dependencies
- `#include "develop.h"` — provides build configuration macros (`SHAREWARE`, `DELUXE`, `LOWCOST`, `SUPERROTT`) that determine WAD checksum at compile time
