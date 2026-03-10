# rtsmaker/cmdlib.c
## File Purpose
Command-line utility library providing core infrastructure for the RTS maker tool. Implements safe file I/O, command-line argument parsing, path manipulation, memory management, number parsing, byte-order conversion, and VGA graphics palette operations. Compiled with conditional platform support for DOS/Windows and NeXTStep/Unix.

## Core Responsibilities
- Command-line argument parsing and validation
- Safe file operations with error handling and chunked I/O (up to 32KB per chunk)
- Safe memory allocation with error reporting
- Path string manipulation (extensions, base names, directory stripping)
- Number parsing (decimal and hexadecimal formats)
- Byte-order conversion (big-endian ↔ little-endian)
- VGA palette read/write operations via I/O ports
- Cross-platform abstraction (DOS vs. NeXTStep)

## External Dependencies
- **Standard C library**: `<process.h>`, `<io.h>`, `<dos.h>`, `<stdlib.h>`, `<stdio.h>`, `<stdarg.h>`, `<ctype.h>`, `<string.h>`, `<fcntl.h>`, `<sys/stat.h>`, `<conio.h>` (DOS); `<libc.h>`, `<errno.h>` (NeXT).
- **Platform-specific syscalls**: `open()`, `read()`, `write()`, `close()` (POSIX), `fstat()` (NeXT), `inp()`/`outp()` (DOS VGA I/O).
- **Local**: `cmdlib.h` (header with function declarations and type definitions).
- **Defined elsewhere**: `myargc`, `myargv` (or `NXargc`/`NXargv`, `_argc`/`_argv` depending on platform).

# rtsmaker/cmdlib.h
## File Purpose
Declares utility functions for build and command-line tools. Provides foundational I/O, memory, file operations, and system mode switching for Apogee's development pipeline.

## Core Responsibilities
- Command-line argument parsing and error reporting
- Safe file I/O with error handling (open, read, write, load, save)
- Memory management with safety wrappers (allocate, deallocate)
- File path manipulation (default extensions, base paths, filename extraction)
- Endianness conversion for cross-platform binary format handling
- Display/palette management (VGA mode, text mode, palette I/O)

## External Dependencies
- Conditional `strcasecmp` compatibility for NeXT systems
- No standard library includes visible in header (implementations import separately)
- Platform-specific video/palette hardware access (DOS VGA assumed)

# rtsmaker/rtsmaker.c
## File Purpose
RTSMAKER is a utility that packages remote player sound files (VOC/WAV format) into RTS container files for Duke Nukem 3D and Rise of the Triad multiplayer modes. It reads a script file specifying an output RTS filename and up to 10 sound files, then creates a WAD-format archive. It also supports unpacking existing RTS files back to individual sound files.

## Core Responsibilities
- Parse script files containing RTS configuration and sound file references
- Read and copy sound file data (VOC/WAV files) into a WAD-format container
- Manage lump (sound chunk) metadata and directory structure
- Write RTS file headers and lump tables
- Support unpacking mode to extract lumps from existing RTS files
- Validate that exactly 10 sounds are provided per script

## External Dependencies
- **cmdlib.h**: `SafeOpenRead()`, `SafeOpenWrite()`, `SafeRead()`, `SafeWrite()`, `SafeMalloc()`, `SafeFree()`, `Error()`, `CheckParm()`, `DefaultPath()`, `DefaultExtension()`, `ExtractFileBase()`, `LittleLong()`
- **scriplib.h**: `LoadScriptFile()`, `GetToken()`, global `token[]`, global `endofscript` flag
- **Standard C**: `stdlib.h`, `stdio.h`, `string.h` (file I/O, memory, string utilities)
- **Win32-specific**: `<io.h>`, `<fcntl.h>`, `<direct.h>` (file handles, getcwd, close, unlink)

# rtsmaker/scriplib.c
## File Purpose
Provides a tokenizing script parser for reading configuration/script files. Loads entire scripts into memory and extracts whitespace-delimited tokens with support for comments and line tracking.

## Core Responsibilities
- Load script files from disk into memory (`LoadScriptFile`)
- Extract and buffer individual tokens from script content
- Track parsing state (current position, line number, end-of-file)
- Skip whitespace, newlines, and semicolon-delimited comments
- Support token lookahead/pushback via `UnGetToken`
- Validate line completeness when `crossline=false`

## External Dependencies
- **Includes:** `cmdlib.h` (error handling, file I/O), platform-specific: `libc.h` (NeXT/Unix), `io.h`/`dos.h`/`fcntl.h` (DOS/Windows)
- **Defined elsewhere:** `Error()`, `LoadFile()` (from cmdlib)
- **Types from cmdlib.h:** `boolean`, `byte`

# rtsmaker/scriplib.h
## File Purpose
Header file for a script parsing library used in the RTS tool. Provides a token-based lexer interface to load and parse script files from disk into memory, with functions to retrieve tokens sequentially and track parsing state.

## Core Responsibilities
- Load script files into memory buffers
- Tokenize script content by extracting tokens separated by whitespace/delimiters
- Manage parsing state (current position, script boundaries, line tracking)
- Provide token retrieval and ungetting (pushback) operations
- Track EOF and parsing completion status

## External Dependencies
- `cmdlib.h` – provides basic types (`byte`, `boolean`) and utility functions (`Error`, memory/file I/O)
- Global state and function implementations defined elsewhere (not in this header)

