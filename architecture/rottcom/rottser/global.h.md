# rottcom/rottser/global.h

## File Purpose
Global header for the ROTT serialization/server component. Provides fundamental constants, type definitions, and declarations for utility functions handling error reporting, file I/O, memory management, and string operations.

## Core Responsibilities
- Define basic constants (TRUE, FALSE, clock frequency, ESC character)
- Define low-level I/O and interrupt control macros (INPUT, OUTPUT, CLI, STI)
- Define core data types (boolean, byte, word, longword, fixed-point)
- Declare error handling and fatal exit functions
- Declare file I/O utilities with error handling
- Declare safe memory allocation and string manipulation functions
- Declare command-line parameter parsing utilities

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| boolean | enum | Boolean type with false/true values |
| byte | typedef | Unsigned 8-bit integer |
| word | typedef | Unsigned 16-bit integer |
| longword | typedef | Unsigned 32-bit integer |
| fixed | typedef | Fixed-point signed 32-bit number |

## Global / File-Static State
None.

## Key Functions / Methods

### Error
- Signature: `void Error (char *error, ...)`
- Purpose: Report fatal error and terminate program
- Inputs: Format string and variadic arguments
- Outputs/Return: None (program exit)
- Side effects: Prints error message, exits process
- Notes: Variadic function suggests printf-like formatting

### SafeOpenRead / SafeOpenWrite
- Signature: `int SafeOpenRead (char *filename)` / `int SafeOpenWrite (char *filename)`
- Purpose: Open file for reading/writing with automatic error handling
- Inputs: Filename
- Outputs/Return: File handle
- Side effects: Opens file; calls Error() on failure
- Notes: "Safe" prefix indicates errors trigger fatal exit, not NULL return

### SafeRead / SafeWrite / SafeWriteString
- Signature: `void SafeRead (int handle, void *buffer, long count)` / `void SafeWrite (int handle, void *buffer, long count)` / `void SafeWriteString (int handle, char *buffer)`
- Purpose: Read/write data to file with error handling
- Inputs: File handle, buffer pointer, byte count
- Outputs/Return: None (data read/written in place)
- Side effects: File I/O; calls Error() on failure
- Notes: SafeWriteString specializes for null-terminated strings

### LoadFile / SaveFile
- Signature: `long LoadFile (char *filename, void **bufferptr)` / `void SaveFile (char *filename, void *buffer, long count)`
- Purpose: Load entire file into allocated memory / save buffer to file
- Inputs: Filename; LoadFile uses pointer-to-pointer for output buffer
- Outputs/Return: LoadFile returns file size; SaveFile returns void
- Side effects: Heap allocation (LoadFile), file creation (SaveFile)

### SafeMalloc
- Signature: `void *SafeMalloc (long size)`
- Purpose: Allocate heap memory with error handling
- Inputs: Size in bytes
- Outputs/Return: Void pointer to allocated block
- Side effects: Allocates heap; calls Error() if allocation fails (never returns NULL)

### ExtractFileBase
- Signature: `void ExtractFileBase (char *path, char *dest)`
- Purpose: Extract filename from path (strip directory and extension)
- Inputs: Full file path, destination buffer
- Outputs/Return: None (fills dest)
- Side effects: None

### CheckParm / US_CheckParm
- Signature: `int CheckParm (char *check)` / `int US_CheckParm (char *parm, char **strings)`
- Purpose: Parse command-line arguments; CheckParm checks existence, US_CheckParm matches against string array
- Inputs: Parameter name; string array (for US_CheckParm)
- Outputs/Return: Index or -1 if not found
- Notes: US prefix suggests "Utility String"

### ReadParameter / WriteParameter
- Signature: `void ReadParameter (const char *s1, int *val)` / `void WriteParameter (int file, const char *s1, int val)`
- Purpose: Read/write configuration parameters (likely to/from config file)
- Inputs: Parameter name, integer value
- Outputs/Return: None (ReadParameter fills val by reference)

### Utility Functions
- `StringLength`: Get length of null-terminated string
- `UL_strcpy`: Safe string copy with size limit (byte-level)
- `UL_DisplayMemoryError`: Display memory error message (UL = "Utility Library" prefix)

## Control Flow Notes
Header-only; provides utilities called throughout application lifecycle: parameter parsing at startup, file I/O during runtime, memory management throughout, and error handling at shutdown/failure points.

## External Dependencies
- Low-level I/O macros wrap functions defined elsewhere: `inp()`, `outp()`, `disable()`, `enable()` (DOS/x86-specific port I/O and interrupt control)
- No explicit #includes visible; likely includes standard C library headers elsewhere
