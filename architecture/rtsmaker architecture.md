# Subsystem Overview

## Purpose

RTSMAKER is a build utility that packages remote player sound files (VOC/WAV format) into RTS container files for Duke Nukem 3D and Rise of the Triad multiplayer modes. It reads script configurations and creates WAD-format archives containing sound lumps, supporting both pack and unpack operations for remote player audio delivery.

## Key Files

| File | Role |
|------|------|
| `cmdlib.c` / `cmdlib.h` | Cross-platform file I/O, memory management, command-line parsing, byte-order conversion, VGA palette operations |
| `rtsmaker.c` | Main tool logic: script parsing, sound file packaging into RTS/WAD containers, lump metadata management |
| `scriplib.c` / `scriplib.h` | Token-based script file parser with whitespace/comment handling and state tracking |

## Core Responsibilities

- Parse script files specifying RTS output filename and up to 10 VOC/WAV sound file references
- Read and copy sound file data into WAD-format container structures
- Manage lump metadata and WAD directory tables for packaged sounds
- Write RTS file headers and serialize lump tables to disk
- Unpack existing RTS files, extracting individual sound lumps back to files
- Provide cross-platform file I/O abstractions (DOS/Windows and NeXTStep/Unix)
- Handle command-line argument parsing and validation
- Perform safe file operations with error reporting and chunked I/O (up to 32KB per chunk)

## Key Interfaces & Data Flow

**Inputs:**
- Script files (text format specifying RTS output path and sound file list)
- VOC/WAV sound files (binary audio data)
- Optional command-line flags for pack/unpack operations

**Outputs:**
- RTS container files (WAD-format archives with sound lumps and metadata)
- Individual sound files (when unpacking mode is used)

**Subsystem interactions:**
- Uses `cmdlib` utilities for safe file operations, memory allocation, and platform abstraction
- Uses `scriplib` to tokenize and parse configuration scripts
- No direct dependency on game runtime code; pure utility/build-time tool

## Runtime Role

Not inferable from provided docs. As a command-line build utility, RTSMAKER executes on-demand during asset packaging phases (prior to multiplayer deployment), not within the game engine's init/frame/shutdown cycle.

## Notable Implementation Details

- Inherits WAD-format container structure from DOOM heritage (consistent with RT_* asset pipeline conventions)
- Enforces exactly 10 sounds per script validation (fixed array requirement for RTS format)
- Token-based script parsing with semicolon-delimited comment stripping and line tracking
- Conditional compilation for platform-specific code paths (DOS VGA I/O vs. NeXTStep system calls)
- Watcom C v10.0b compatibility with platform-specific headers (`<dos.h>`, `<conio.h>` for DOS; `<libc.h>` for NeXT)
