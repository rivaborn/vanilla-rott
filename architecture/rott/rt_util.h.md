# rott/rt_util.h

## File Purpose
Utility header declaring functions for palette management, file I/O, memory allocation, graphics operations, and hardware-level port I/O. Core support layer for the ROTT game engine providing safe wrappers around system resources and utility helpers for color, math, and file path handling.

## Core Responsibilities
- Palette acquisition, modification, and EGA color mapping
- Safe file I/O operations with error handling
- Memory allocation with level-based tracking
- File path parsing and manipulation
- Graphics screen and buffer operations
- Hardware port I/O for graphics mode setup
- String/number parsing and byte-order conversions
- Math utilities (distance, angle approximation)
- Error reporting and debug output
- Command-line parameter checking

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `egacolor` | `int[16]` | extern global | EGA palette color table |
| `origpal` | `byte*` | extern global | Pointer to original palette data |
| `_argc` | `int` | extern global | Command-line argument count |
| `_argv` | `char**` | extern global | Command-line argument vector |

## Key Functions / Methods

### SafeOpenRead, SafeOpenWrite, SafeOpenAppend
- **Signature:** `int SafeOpenRead(char *filename)`, etc.
- **Purpose:** Safe file opening with error handling for read/write/append modes
- **Inputs:** filename (null-terminated string)
- **Outputs/Return:** File handle (int); errors trigger Error() or SoftwareError()
- **Side effects:** File descriptor allocation; may trigger error handlers
- **Calls:** Error(), SoftwareError() (implied)
- **Notes:** Paired with SafeRead/SafeWrite for bounded I/O

### SafeMalloc, SafeLevelMalloc
- **Signature:** `void *SafeMalloc(long size)`, `void *SafeLevelMalloc(long size)`
- **Purpose:** Allocate memory with error checking; SafeLevelMalloc tracks level-specific allocations
- **Inputs:** size (bytes)
- **Outputs/Return:** Pointer to allocated memory or error handler invoked
- **Side effects:** Memory allocation; may trigger SoftwareError() or UL_DisplayMemoryError()
- **Calls:** Error(), SoftwareError() (implied)
- **Notes:** Inverse is SafeFree(); level-based allocation used for cleanup at level/map boundaries

### GetPalette, SetPalette, VL_SetPalette
- **Signature:** `void GetPalette(char *pal)`, `void SetPalette(char *pal)`, `void VL_SetPalette(byte *palette)`
- **Purpose:** Retrieve or apply color palettes; SetaPalette and VL variants differ in parameter type/target
- **Inputs:** pal/palette pointer (byte or char array, typically 768 bytes for 256×3 RGB)
- **Outputs/Return:** None (void)
- **Side effects:** Modifies hardware palette or internal state
- **Calls:** Hardware port operations (implied via pragma aux)
- **Notes:** FindEGAColors() maps arbitrary colors to EGA palette

### LoadFile, SaveFile
- **Signature:** `long LoadFile(char *filename, void **bufferptr)`, `void SaveFile(char *filename, void *buffer, long count)`
- **Purpose:** Load entire file into allocated buffer; save buffer to file
- **Inputs:** filename, bufferptr (output), buffer (input), count (bytes)
- **Outputs/Return:** LoadFile returns file size; bufferptr allocated by SafeMalloc
- **Side effects:** Memory allocation (LoadFile); file I/O
- **Calls:** SafeOpenRead/Write, SafeRead/Write, SafeMalloc
- **Notes:** LoadFile allocates; caller responsible for SafeFree

### ExtractFileBase, DefaultExtension, DefaultPath
- **Signature:** `void ExtractFileBase(char *path, char *dest)`, etc.
- **Purpose:** Parse/manipulate file paths; apply defaults if missing
- **Inputs:** path, dest/extension/basepath
- **Outputs/Return:** Outputs via dest pointer (void functions)
- **Side effects:** String buffer modification
- **Calls:** None visible
- **Notes:** Trivial string manipulation helpers

### FindDistance, Find_3D_Distance, atan2_appx
- **Signature:** `int FindDistance(int ix, int iy)`, `int Find_3D_Distance(int ix, int iy, int iz)`, `int atan2_appx(int, int)`
- **Purpose:** Compute Euclidean distance (2D/3D); fast approximate arctangent
- **Inputs:** Coordinates (integers)
- **Outputs/Return:** Distance or angle approximation (int)
- **Side effects:** None
- **Calls:** None visible
- **Notes:** Performance-critical for collision/visibility; approximations trade accuracy for speed

### ParseHex, ParseNum
- **Signature:** `long ParseHex(char *hex)`, `long ParseNum(char *str)`
- **Purpose:** Convert string to hexadecimal or decimal number
- **Inputs:** String (hex/decimal)
- **Outputs/Return:** long value
- **Side effects:** None
- **Calls:** None visible

### MotoShort, IntelShort, MotoLong, IntelLong
- **Signature:** `short MotoShort(short l)`, etc.
- **Purpose:** Byte-order conversion (Motorola ↔ Intel endianness)
- **Inputs:** 16-bit or 32-bit integer
- **Outputs/Return:** Converted value
- **Side effects:** None
- **Calls:** None visible
- **Notes:** Legacy multi-platform support (68k vs x86)

### Square, my_outp (pragma aux)
- **Signature:** `void Square(void)`, `void my_outp(int port, int data)`
- **Purpose:** Hardware operations via inline assembly; Square sets graphics mode, my_outp writes port
- **Inputs:** port (edx), data (eax)
- **Outputs/Return:** None (void)
- **Side effects:** Direct hardware I/O
- **Calls:** None (raw assembly)
- **Notes:** `#pragma aux` directives embed x86 assembly; OUTP is macro alias for my_outp

### Error, SoftwareError, DebugError
- **Signature:** `void Error(char *error, ...)`, `void SoftwareError(...)`, `void DebugError(...)`
- **Purpose:** Error reporting; SoftwareError/DebugError conditional on compile flags
- **Inputs:** Format string + variadic args (printf-style)
- **Outputs/Return:** None (void)
- **Side effects:** Prints to screen/log; may abort or enter soft-error handler
- **Calls:** Internal handlers (StartupSoftError, ShutdownSoftError)
- **Notes:** SoftError and Debug are macro wrappers; DEBUG and SOFTERROR flags control behavior

## Control Flow Notes
This header is a utility library with no primary control flow. Functions are called on-demand throughout the engine: file I/O during level loading, memory allocation during initialization/level transitions, palette setup during graphics init, and math utilities during gameplay (collision, visibility). The `Square()` and `my_outp()` functions are called once during graphics mode setup (likely in main init); error handlers may interrupt normal flow by entering a soft-error loop.

## External Dependencies
- **Local includes:** `develop.h` (build flags: DEBUG, SOFTERROR, TEXTMENUS, SHAREWARE, SUPERROTT, etc.)
- **Implied external symbols:** Standard C file I/O (open, read, write, close), malloc/free, printf/sprintf-style functions, hardware graphics mode setup (VGA port writes)
- **Language features:** `#pragma aux` (Watcom C inline assembly); variadic functions (`...`)
