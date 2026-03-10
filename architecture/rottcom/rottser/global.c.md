# rottcom/rottser/global.c

## File Purpose

Utility library providing safe wrappers for file I/O, memory allocation, and error handling for the ROTT multiplayer server setup system. Includes configuration file parameter reading/writing and string manipulation helpers.

## Core Responsibilities

- Error reporting with variadic formatting and program termination
- Command-line argument parsing
- Safe file operations (open, read, write) with I/O error checking and chunking
- Error-checked memory allocation
- Configuration parameter serialization/deserialization from script format
- String length calculation and bounded string copying

## Key Types / Data Structures

None.

## Global / File-Static State

None.

## Key Functions / Methods

### Error
- **Signature:** `void Error(char *error, ...)`
- **Purpose:** Formatted error reporting and abnormal program termination
- **Inputs:** Format string and variadic arguments
- **Outputs/Return:** None (terminates program)
- **Side effects:** Prints to stdout, calls `ShutDown()`, calls `exit()`
- **Calls:** `va_start()`, `vprintf()`, `va_end()`, `ShutDown()`, `exit()`
- **Notes:** Exit code is 0 if `error == NULL`, 1 otherwise

### CheckParm
- **Signature:** `int CheckParm(char *check)`
- **Purpose:** Search command-line arguments for a matching parameter
- **Inputs:** Parameter string to match
- **Outputs/Return:** Index (1 to argc-1) if found, 0 if not found
- **Side effects:** None
- **Calls:** `stricmp()`
- **Notes:** Uses global `_argc` and `_argv` from C runtime

### SafeOpenWrite
- **Signature:** `int SafeOpenWrite(char *filename)`
- **Purpose:** Open file for reading/writing with truncation
- **Inputs:** Filename
- **Outputs/Return:** File handle (int)
- **Side effects:** Creates or truncates file; calls `Error()` on failure
- **Calls:** `open()`, `Error()`
- **Notes:** Uses `O_RDWR | O_BINARY | O_CREAT | O_TRUNC`

### SafeOpenRead
- **Signature:** `int SafeOpenRead(char *filename)`
- **Purpose:** Open file for reading
- **Inputs:** Filename
- **Outputs/Return:** File handle (int)
- **Side effects:** Calls `Error()` on failure
- **Calls:** `open()`, `Error()`

### SafeRead
- **Signature:** `void SafeRead(int handle, void *buffer, long count)`
- **Purpose:** Read data from file, chunking large reads into 32KB blocks
- **Inputs:** File handle, buffer pointer, byte count
- **Outputs/Return:** None (buffer modified in-place)
- **Side effects:** I/O, calls `Error()` on read failure
- **Calls:** `read()`, `Error()`
- **Notes:** Reads in 0x8000 (32KB) chunks; terminates on short read

### SafeWrite
- **Signature:** `void SafeWrite(int handle, void *buffer, long count)`
- **Purpose:** Write data to file, chunking large writes into 32KB blocks
- **Inputs:** File handle, buffer pointer, byte count
- **Outputs/Return:** None
- **Side effects:** I/O, calls `Error()` on write failure
- **Calls:** `write()`, `Error()`

### SafeWriteString
- **Signature:** `void SafeWriteString(int handle, char *buffer)`
- **Purpose:** Write null-terminated string to file
- **Inputs:** File handle, string
- **Outputs/Return:** None
- **Side effects:** I/O, calls `Error()` on write failure
- **Calls:** `strlen()`, `write()`, `Error()`

### SafeMalloc
- **Signature:** `void *SafeMalloc(long size)`
- **Purpose:** Allocate memory with error checking
- **Inputs:** Size in bytes
- **Outputs/Return:** Pointer to allocated memory
- **Side effects:** Calls `Error()` if allocation fails
- **Calls:** `malloc()`, `Error()`

### LoadFile
- **Signature:** `long LoadFile(char *filename, void **bufferptr)`
- **Purpose:** Read entire file into dynamically allocated buffer
- **Inputs:** Filename, output buffer pointer
- **Outputs/Return:** File size in bytes
- **Side effects:** Allocates memory, modifies `*bufferptr`
- **Calls:** `SafeOpenRead()`, `filelength()`, `SafeMalloc()`, `SafeRead()`, `close()`

### ReadParameter
- **Signature:** `void ReadParameter(const char *s1, int *val)`
- **Purpose:** Parse integer parameter from script token stream
- **Inputs:** Parameter name, pointer to output int
- **Outputs/Return:** None (result via `*val`)
- **Side effects:** Advances token stream, calls `Error()` if parameter not found
- **Calls:** `GetToken()`, `Error()`, `strcmpi()`, `atoi()`
- **Notes:** Uses globals `token` and `endofscript` from scriplib; searches until parameter found or EOF

### WriteParameter
- **Signature:** `void WriteParameter(int file, const char *s1, int val)`
- **Purpose:** Write integer parameter to configuration file
- **Inputs:** File handle, parameter name, integer value
- **Outputs/Return:** None
- **Side effects:** I/O
- **Calls:** `SafeWriteString()`, `strcpy()`, `itoa()`
- **Notes:** Format: "paramname  value\n"

### StringLength
- **Signature:** `int StringLength(char *string)`
- **Purpose:** Calculate string length including null terminator
- **Inputs:** String pointer
- **Outputs/Return:** Length in bytes (including null)
- **Side effects:** None
- **Calls:** None
- **Notes:** Redundant with standard `strlen() + 1`

### UL_strcpy
- **Signature:** `void UL_strcpy(byte *dest, byte *src, int size)`
- **Purpose:** Bounded string copy
- **Inputs:** Destination, source, maximum size
- **Outputs/Return:** None (destination modified in-place)
- **Side effects:** None
- **Calls:** None
- **Notes:** Null-terminates destination; **contains bug**: `size++` should be `size--` (increments instead of decrements)

## Control Flow Notes

Utility library called during initialization phase (config file parsing via `ReadParameter`) and file loading (via `LoadFile`). Configuration writing occurs at unspecified points. Error handler acts as fatal error sink from any module.

## External Dependencies

- **Standard C library:** `malloc.h`, `fcntl.h`, `errno.h`, `stdlib.h`, `stdio.h`, `string.h`, `stdarg.h`, `sys/stat.h`
- **DOS/Windows legacy:** `conio.h`, `io.h`, `dos.h`, `dir.h` (indicates 16-bit DOS/Windows code)
- **Local headers:** `sersetup.h` (calls `ShutDown()`), `scriplib.h` (uses `GetToken()`, `token`, `endofscript`)
- **External symbols:** `_argc`, `_argv` (C runtime), `strerror()`, `open()`, `read()`, `write()`, `close()`, `filelength()`, `strlen()`, `strcpy()`, `itoa()`, `atoi()`, `stricmp()`, `strcmpi()`
