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

## Key Types / Data Structures
None (defines `byte` and `boolean` in header; this file only implements functions).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `myargc` / `myargv` | `int`, `char**` | extern (conditional) | Command-line argument count and vector, conditionally mapped to `NXargc`/`NXargv` (NeXT) or `_argc`/`_argv` (Watcom). |
| `PATHSEPERATOR` | char macro | conditional | Platform-specific path separator (`\\` DOS, `/` NeXT). |
| `O_BINARY` | int macro | conditional | File open flag, 0 on NeXT, real flag on DOS. |

## Key Functions / Methods

### Error
- Signature: `void Error(char *error, ...)`
- Purpose: Abnormal termination with formatted error message (variadic printf-style).
- Inputs: Format string and variadic arguments.
- Outputs/Return: None (terminates process with `exit(1)`).
- Side effects: Prints to stdout via `vprintf`, exits process.
- Calls: `va_start`, `vprintf`, `va_end`, `printf`, `exit`.
- Notes: Always terminates; used throughout library for fatal errors.

### CheckParm
- Signature: `int CheckParm(char *check)`
- Purpose: Search command-line arguments for a given parameter.
- Inputs: Parameter name string (e.g., "−o", "−x").
- Outputs/Return: Argument index (1 to argc-1) or 0 if not found.
- Side effects: None.
- Calls: `isalpha`, `stricmp`.
- Notes: Skips leading non-alphabetic characters (−, /, \). Case-insensitive comparison.

### SafeOpenWrite
- Signature: `int SafeOpenWrite(char *filename)`
- Purpose: Open file for writing with exclusive creation.
- Inputs: File path.
- Outputs/Return: File handle, or terminates via `Error()` on failure.
- Side effects: Creates/truncates file, I/O.
- Calls: `open`, `Error`.
- Notes: Uses `O_RDWR | O_BINARY | O_CREAT | O_TRUNC` flags; sets read/write permissions.

### SafeOpenRead
- Signature: `int SafeOpenRead(char *filename)`
- Purpose: Open file for reading.
- Inputs: File path.
- Outputs/Return: File handle, or terminates via `Error()` on failure.
- Side effects: I/O.
- Calls: `open`, `Error`.

### SafeRead / SafeWrite
- Signature: `void SafeRead(int handle, void *buffer, long count)` / `void SafeWrite(int handle, void *buffer, long count)`
- Purpose: Chunked file I/O (max 32KB per system call) to handle large reads/writes.
- Inputs: File handle, buffer pointer, byte count.
- Outputs/Return: None.
- Side effects: I/O, modifies buffer for read, advances buffer pointer internally.
- Calls: `read`/`write`, `Error`.
- Notes: Loops in 32KB (`0x8000`) chunks; terminates via `Error()` on partial I/O.

### SafeMalloc / SafeFree
- Signature: `void *SafeMalloc(long size)` / `void SafeFree(void *ptr)`
- Purpose: Wrapped malloc with error checking / standard free.
- Inputs: Allocation size / pointer to free.
- Outputs/Return: Allocated pointer / none.
- Side effects: Memory allocation/deallocation.
- Calls: `malloc`, `Error` (SafeMalloc) / `free` (SafeFree).

### LoadFile / SaveFile
- Signature: `long LoadFile(char *filename, void **bufferptr)` / `void SaveFile(char *filename, void *buffer, long count)`
- Purpose: Complete file read/write operations.
- Inputs: Filename, output buffer pointer (LoadFile) / input buffer and byte count (SaveFile).
- Outputs/Return: File size (LoadFile) / none.
- Side effects: I/O, memory allocation (LoadFile).
- Calls: `SafeOpenRead`/`SafeOpenWrite`, `filelength`, `SafeMalloc`, `SafeRead`/`SafeWrite`, `close`.
- Notes: `LoadFile` allocates buffer and sets via pointer; caller must free.

### DefaultExtension / DefaultPath
- Signature: `void DefaultExtension(char *path, char *extension)` / `void DefaultPath(char *path, char *basepath)`
- Purpose: Append default file extension if missing / prepend base path if relative.
- Inputs: Path string, extension (with dot) / base path.
- Outputs/Return: None (in-place modification).
- Side effects: String concatenation.
- Calls: `strlen`, `strcat`, `strcpy`.
- Notes: `DefaultExtension` scans backward for `\` or `.`; skips if extension found.

### StripFilename / ExtractFileBase
- Signature: `void StripFilename(char *path)` / `void ExtractFileBase(char *path, char *dest)`
- Purpose: Remove filename (keep directory) / extract base name (up to 8 chars, uppercase).
- Inputs: Path / destination buffer.
- Outputs/Return: None (in-place for StripFilename, dest filled for ExtractFileBase).
- Side effects: String manipulation.
- Calls: `strlen`, `memset`, `toupper`, `Error` (ExtractFileBase on overflow).
- Notes: `ExtractFileBase` limits to 8 chars (DOS 8.3 convention); errors if longer.

### ParseHex / ParseNum
- Signature: `long ParseHex(char *hex)` / `long ParseNum(char *str)`
- Purpose: Parse hexadecimal or generic numeric strings.
- Inputs: Hex string / numeric string (may have `$` or `0x` prefix).
- Outputs/Return: Parsed long value.
- Side effects: None.
- Calls: `ParseHex`, `atol`, `Error` (ParseHex on invalid char).
- Notes: `ParseNum` dispatches to `ParseHex` for `$` or `0x` prefixes.

### BigShort / BigLong / LittleShort / LittleLong
- Signature: `short BigShort(short l)`, `long BigLong(long l)`, `short LittleShort(short l)`, `long LittleLong(long l)`
- Purpose: Byte-order conversion for multi-byte integers.
- Inputs: 16-bit or 32-bit integer.
- Outputs/Return: Converted integer (or unchanged for little-endian on little-endian platform).
- Side effects: None.
- Calls: None (bit operations only).
- Notes: Assumes native little-endian architecture; `LittleShort`/`LittleLong` are no-ops (return input unchanged).

### GetPalette / SetPalette
- Signature: `void GetPalette(byte *pal)` / `void SetPalette(byte *pal)`
- Purpose: Read/write 8-bit color palette from/to VGA hardware.
- Inputs: 768-byte palette buffer (256 colors × 3 bytes RGB).
- Outputs/Return: None (palette filled or written to I/O ports).
- Side effects: Hardware I/O (VGA palette registers).
- Calls: `outp`, `inp` (DOS only; no-op on NeXT).
- Notes: Only compiled on non-NeXT platforms. `GetPalette` reads and left-shifts by 2 bits; `SetPalette` right-shifts before write. I/O ports: `PEL_READ_ADR` (0x3c7), `PEL_WRITE_ADR` (0x3c8), `PEL_DATA` (0x3c9).

## Control Flow Notes
This is a utility library—no single initialization/frame/render loop. Typically called from tool startup (command-line parsing), file loading phases (LoadFile), and resource setup (GetPalette for VGA mode initialization). Error handling centralizes all failures through `Error()`, which terminates the tool immediately.

## External Dependencies
- **Standard C library**: `<process.h>`, `<io.h>`, `<dos.h>`, `<stdlib.h>`, `<stdio.h>`, `<stdarg.h>`, `<ctype.h>`, `<string.h>`, `<fcntl.h>`, `<sys/stat.h>`, `<conio.h>` (DOS); `<libc.h>`, `<errno.h>` (NeXT).
- **Platform-specific syscalls**: `open()`, `read()`, `write()`, `close()` (POSIX), `fstat()` (NeXT), `inp()`/`outp()` (DOS VGA I/O).
- **Local**: `cmdlib.h` (header with function declarations and type definitions).
- **Defined elsewhere**: `myargc`, `myargv` (or `NXargc`/`NXargv`, `_argc`/`_argv` depending on platform).
