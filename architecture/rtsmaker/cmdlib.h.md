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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `byte` | typedef | Unsigned 8-bit value (alias for `unsigned char`) |
| `boolean` | enum | Two-state type with values `{false, true}` |

## Global / File-Static State
None.

## Key Functions / Methods

### Error
- Signature: `void Error(char *error, ...)`
- Purpose: Report fatal errors with formatted message
- Inputs: Format string and variadic arguments
- Outputs/Return: None (typically exits)
- Side effects: Prints to output, terminates execution
- Notes: Standard varargs error handler

### CheckParm
- Signature: `int CheckParm(char *check)`
- Purpose: Search command-line arguments for a parameter
- Inputs: Parameter name to check
- Outputs/Return: Index if found, or negative value if absent
- Side effects: None
- Calls: Implies access to global command-line state

### SafeOpenRead / SafeOpenWrite
- Signature: `int SafeOpenRead(char *filename)`, `int SafeOpenWrite(char *filename)`
- Purpose: Protected file open with error checking
- Inputs: File path
- Outputs/Return: File handle (or error exit)
- Side effects: Opens file, exits on failure

### SafeRead / SafeWrite
- Signature: `void SafeRead(int handle, void *buffer, long count)`, `void SafeWrite(int handle, void *buffer, long count)`
- Purpose: Guarded file I/O ensuring full operation
- Inputs: File handle, buffer, byte count
- Outputs/Return: None (exits on error)
- Side effects: Transfers data between file and memory

### SafeMalloc / SafeFree
- Signature: `void *SafeMalloc(long size)`, `void SafeFree(void *ptr)`
- Purpose: Protected memory allocation/deallocation
- Inputs: Allocation size / pointer to free
- Outputs/Return: Allocated pointer / none
- Side effects: Allocates or frees heap memory; exits on allocation failure

### LoadFile / SaveFile
- Signature: `long LoadFile(char *filename, void **bufferptr)`, `void SaveFile(char *filename, void *buffer, long count)`
- Purpose: Convenience wrappers for loading/saving entire files
- Inputs: Filename, buffer (and size for SaveFile)
- Outputs/Return: File size (LoadFile) / none
- Side effects: I/O, memory allocation (LoadFile)

### Path Manipulation (DefaultExtension, DefaultPath, StripFilename, ExtractFileBase)
- Purpose: String-based path normalization and parsing
- Notes: Operate on in-place path buffers; no bounds checking visible

### Endianness Conversion (BigShort, LittleShort, BigLong, LittleLong)
- Purpose: Convert 16-bit and 32-bit values between byte orders
- Inputs: Numeric value
- Outputs/Return: Converted value
- Notes: Essential for portable binary file formats

### GetPalette / SetPalette
- Signature: `void GetPalette(byte *pal)`, `void SetPalette(byte *pal)`
- Purpose: Retrieve/apply 256-color palette to display
- Inputs: Palette buffer (768 bytes typical)
- Side effects: Hardware palette I/O

### VGAMode / TextMode
- Signature: `void VGAMode(void)`, `void TextMode(void)`
- Purpose: Switch display mode for DOS/early 90s environments
- Side effects: Change video hardware state

## Control Flow Notes
Tool-phase utility library, not game engine runtime. Used during asset loading, file conversion, and build initialization. Display mode switching suggests DOS executable context.

## External Dependencies
- Conditional `strcasecmp` compatibility for NeXT systems
- No standard library includes visible in header (implementations import separately)
- Platform-specific video/palette hardware access (DOS VGA assumed)
