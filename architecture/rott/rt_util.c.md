# rott/rt_util.c

## File Purpose
Utility module providing low-level services for the ROTT engine: palette/color management, safe file I/O with error handling, memory allocation wrappers, mathematical approximations (distance, angle), path/string parsing, and debug logging.

## Core Responsibilities
- Palette and color lookup (EGA color mapping, RGB-to-palette quantization, gamma correction)
- Safe file operations with chunking for large files (32KB threshold)
- Zone memory allocation wrapper with error checking
- Mathematical approximations (2D/3D distance, arctangent via octant lookup)
- Command-line argument parsing and number format conversion
- Debug/error logging to files (with conditional compilation gates)
- Direct video memory text output and graphics primitives
- DOS-style path/drive navigation
- Generic heap sort with custom comparison/switch callbacks

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `PFI` | typedef | Pointer to function returning int (used for sort comparator) |
| `PFV` | typedef | Pointer to function returning void (used for sort switcher) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `egacolor[16]` | int array | global | Maps standard EGA palette indices to actual palette colors |
| `origpal` | byte* | global | Pointer to original game palette |
| `errout` | FILE* | global | File handle for soft error logging |
| `debugout` | FILE* | global | File handle for debug output |
| `mapdebugout` | FILE* | global | File handle for map debug info |
| `SoftErrorStarted` | boolean | static | Tracks if error logging file is open |
| `DebugStarted` | boolean | static | Tracks if debug logging file is open |
| `MapDebugStarted` | boolean | static | Tracks if map debug file is open |
| `egargb[48]` | unsigned char array | static | Hardcoded RGB triplets (16 colors × 3 bytes) for EGA palette |
| `ROTT_ERR` | byte* | external | Pre-rendered error screen image from version.h |

## Key Functions / Methods

### Error
- **Signature:** `void Error(char *error, ...)`
- **Purpose:** Fatal error handler; displays error message, logs diagnostic info, and terminates.
- **Inputs:** Format string and variadic arguments (printf-style)
- **Outputs/Return:** None (exits via `exit(1)`)
- **Side effects:** Calls `ShutDown()`, switches to text mode, copies pre-rendered error screen to video memory at 0xB8000, prints player position/angle and game state, appends error screenshot to file via `SafeOpenAppend`, waits for keypress (if sound enabled), then exits.
- **Calls:** `ShutDown()`, `TextMode()`, `memcpy()`, `UL_printf()`, `va_start/vsprintf/va_end()`, `GetToken()` (script parsing), `GetPathFromEnvironment()`, `SafeOpenAppend()`, `SafeWrite()`, `close()`, `printf()`, `getch()`, `exit()`
- **Notes:** Guards against infinite recursion with static `inerror` counter; temporarily hijacks script parser for message tokenization; modifies global `px`, `py` for text positioning.

### FindDistance
- **Signature:** `int FindDistance(int ix, int iy)`
- **Purpose:** Approximate Euclidean distance in 2D space using fast bit-shift arithmetic.
- **Inputs:** ix, iy (coordinates; sign is discarded via abs)
- **Outputs/Return:** Integer approximation of distance
- **Side effects:** None
- **Calls:** `abs()`
- **Notes:** Uses fixed-point approximation: `ix - (ix>>5) - (ix>>7) + (t>>2) + (t>>6)` where `t = iy + (iy>>1)`. Requires `ix >= iy` via `SWAP` macro; no multiplication/division, suitable for real-time distance culling.

### Find_3D_Distance
- **Signature:** `int Find_3D_Distance(int ix, int iy, int iz)`
- **Purpose:** Approximate Euclidean distance in 3D space.
- **Inputs:** ix, iy, iz (coordinates; converted to absolute values)
- **Outputs/Return:** Integer approximation of 3D distance
- **Side effects:** None
- **Calls:** `abs()`
- **Notes:** Swaps to identify largest component; formula uses bit-shift approximation: `ix - (ix>>4) + (t>>2) + (t>>3)` where `t = iy + iz`.

### atan2_appx
- **Signature:** `int atan2_appx(int dx, int dy)`
- **Purpose:** Approximate arctangent via octant-based fixed-point lookup, avoiding trigonometric functions.
- **Inputs:** dx, dy (displacement components)
- **Outputs/Return:** Angle as integer in range [0, FINEANGLES)
- **Side effects:** None
- **Calls:** `FixedDiv2()`, `FixedMul()`
- **Notes:** Computes ratio of shorter to longer component via `FixedDiv2`, then maps to octant (1st–8th) based on quadrant signs. Result is scaled by `ANGLESDIV8` and masked to `FINEANGLES-1`. No divide-by-zero if dx=dy=0 (returns 0).

### BestColor
- **Signature:** `byte BestColor(int r, int g, int b, byte *palette)`
- **Purpose:** Find closest palette color to a given RGB value using weighted Euclidean distance.
- **Inputs:** r, g, b (RGB 0–255), palette (768 bytes: 256 colors × 3 bytes)
- **Outputs/Return:** Palette index (0–255) of best match
- **Side effects:** None
- **Calls:** None (direct arithmetic)
- **Notes:** Weights are `WeightR=3, WeightG=5, WeightB=2` (green dominates human perception). Early-exit on perfect match (distortion=0). Initial best is palette[0]; initial distortion is artificially high to ensure replacement.

### CheckParm
- **Signature:** `int CheckParm(char *check)`
- **Purpose:** Parse command-line arguments to locate a parameter string.
- **Inputs:** check (null-terminated string to match, case-insensitive)
- **Outputs/Return:** Argument index (1-based) or 0 if not found
- **Side effects:** Reads global `_argc`, `_argv`
- **Calls:** `isalpha()`, `_fstricmp()`
- **Notes:** Skips leading non-alpha characters (e.g., `-`, `/`, `\`) in arguments. Returns immediately on match; suitable for simple flag detection (e.g., `CheckParm("debug")`).

### SafeOpenRead / SafeOpenWrite / SafeOpenAppend
- **Signature:** `int SafeOpenRead(char *filename)`, `int SafeOpenWrite(char *filename)`, `int SafeOpenAppend(char *filename)`
- **Purpose:** Open files with error handling; file handles are integers (DOS-style).
- **Inputs:** filename (path string)
- **Outputs/Return:** File handle (≥0) or fatal error
- **Side effects:** Calls `Error()` if open fails; modifies file on write/append
- **Calls:** `open()`, `Error()`
- **Notes:** Read uses `O_RDONLY`, Write uses `O_TRUNC`, Append uses `O_APPEND`. All use `O_BINARY` and request `S_IREAD | S_IWRITE` permissions.

### SafeRead / SafeWrite
- **Signature:** `void SafeRead(int handle, void *buffer, long count)`, `void SafeWrite(int handle, void *buffer, long count)`
- **Purpose:** Read/write files in chunks to avoid 32KB I/O limit.
- **Inputs:** handle (file descriptor), buffer (data pointer), count (byte count)
- **Outputs/Return:** None (fatal error on failure)
- **Side effects:** Modifies buffer contents (SafeRead), modifies file (SafeWrite), advances file pointer; calls `Error()` on read/write failure
- **Calls:** `read()`, `write()`, `Error()`
- **Notes:** Chunks I/O into 0x8000 (32KB) blocks. Increments buffer pointer and decrements count per chunk. No explicit seek; relies on sequential file positioning.

### SafeMalloc / SafeLevelMalloc / SafeFree
- **Signature:** `void *SafeMalloc(long size)`, `void *SafeLevelMalloc(long size)`, `void SafeFree(void *ptr)`
- **Purpose:** Allocate/deallocate memory via zone memory system with error checking.
- **Inputs:** size (bytes), ptr (pointer to free)
- **Outputs/Return:** Allocated pointer or fatal error; SafeFree returns void
- **Side effects:** Calls `Z_Malloc()`, `Z_LevelMalloc()`, `Z_Free()` (defined elsewhere); calls `Error()` if null pointer is freed or allocation fails
- **Calls:** `Z_Malloc()`, `Z_LevelMalloc()`, `Z_Free()`, `Error()`
- **Notes:** SafeMalloc uses `PU_STATIC` tag. SafeLevelMalloc uses same tag but different zone. SafeFree is a guard against double-free (rejects NULL pointer).

### LoadFile / SaveFile
- **Signature:** `long LoadFile(char *filename, void **bufferptr)`, `void SaveFile(char *filename, void *buffer, long count)`
- **Purpose:** Convenience wrappers for loading entire files into memory or writing memory to disk.
- **Inputs:** filename, bufferptr (output pointer for LoadFile), buffer & count (for SaveFile)
- **Outputs/Return:** File size (LoadFile) or void (SaveFile)
- **Side effects:** Allocates memory (LoadFile via SafeMalloc), writes file (SaveFile)
- **Calls:** `SafeOpenRead/Write()`, `filelength()`, `SafeMalloc()`, `SafeRead/Write()`, `close()`
- **Notes:** LoadFile uses `filelength()` to determine size. SaveFile takes explicit count (no length computation).

### ParseNum / ParseHex
- **Signature:** `long ParseNum(char *str)`, `long ParseHex(char *hex)`
- **Purpose:** Convert strings to numbers; ParseNum detects hex prefix (`$` or `0x`).
- **Inputs:** str/hex (string)
- **Outputs/Return:** Parsed long integer or fatal error
- **Side effects:** Calls `Error()` on bad hex digit
- **Calls:** `ParseHex()`, `atol()`, `Error()`
- **Notes:** ParseHex shifts left 4 bits per digit; no range checking (overflow possible). ParseNum is dispatcher for hex detection.

### VL_SetPalette / VL_GetPalette
- **Signature:** `void VL_SetPalette(byte *palette)`, `void VL_GetPalette(byte *palette)`
- **Purpose:** Read/write palette via VGA I/O ports with gamma correction applied (SetPalette only).
- **Inputs:** palette (768-byte RGB array, 0–255 per channel)
- **Outputs/Return:** None
- **Side effects:** Direct port I/O to 0x3c8/0x3c7 (PEL_WRITE_ADR/READ_ADR) and 0x3c9 (PEL_DATA)
- **Calls:** `OUTP()`, `inp()`, `FixedMul()` (indirectly via `gammatable` lookup)
- **Notes:** SetPalette applies `gammatable[(gammaindex<<6) + pal_value]` per byte. GetPalette reads raw DAC values. Palette assumed normalized to 6-bit VGA range; caller handles 8-to-6 bit conversion.

### UL_ChangeDirectory / UL_ChangeDrive
- **Signature:** `boolean UL_ChangeDirectory(char *path)`, `boolean UL_ChangeDrive(char *drive)`
- **Purpose:** Navigate DOS directory structure and change drives.
- **Inputs:** path (e.g., `C:\DIR\SUBDIR`), drive (e.g., `C:`)
- **Outputs/Return:** true (success) or false (failure)
- **Side effects:** Calls `chdir()`, `_dos_setdrive()`, `_dos_getdrive()`
- **Calls:** `UL_GetPath()`, `chdir()`, `UL_ChangeDrive()`, `_dos_setdrive()`, `_dos_getdrive()`
- **Notes:** UL_ChangeDirectory parses drive letter (if present), sets drive, then changes directories component-by-component. UL_ChangeDrive uses DOS int 21h equivalent via Watcom pragma. Returns false if any chdir() fails.

### UL_printf
- **Signature:** `void UL_printf(byte *str)`
- **Purpose:** Write text directly to video memory at VGA mode 0xB8000 (text mode), advancing cursor position.
- **Inputs:** str (null-terminated string)
- **Outputs/Return:** None
- **Side effects:** Modifies global `px` (X position), writes to 0xB8000 + (py*160) + (px<<1); skips control bytes (< 32 and > 0)
- **Calls:** None
- **Notes:** Each character takes 2 bytes (char + attribute). Control bytes are skipped but advance pointer. Assumes `py` (Y position) is pre-set globally.

### hsort
- **Signature:** `void hsort(char *base, int nel, int width, int (*compare)(), void (*switcher)())`
- **Purpose:** Generic heap sort using custom comparison and element-swap callbacks.
- **Inputs:** base (array), nel (count), width (element size in bytes), compare (comparator function pointer), switcher (swap function pointer)
- **Outputs/Return:** None (sorts in-place)
- **Side effects:** Reorders array in-place; maintains static `Comp`, `Switch`, `Width`, `Base` pointers/globals during sort
- **Calls:** `newsift_down()` (internal static function)
- **Notes:** Comparator should return negative/zero/positive (like `strcmp`). Switcher is called for swaps. `Base` is set to `base - width` to support 1-indexed heap algorithm. `newsift_down` is static helper for heap heapification.

### SideOfLine
- **Signature:** `int SideOfLine(int x1, int y1, int x2, int y2, int x3, int y3)`
- **Purpose:** Determine which side of a line a point lies on (geometric test).
- **Inputs:** (x1,y1), (x2,y2) (line endpoints), (x3,y3) (test point)
- **Outputs/Return:** -1 (negative side), 0 (on line), +1 (positive side) via `SGN()` macro
- **Side effects:** None
- **Calls:** `FixedMulShift()`, `SGN()`
- **Notes:** Computes signed distance using cross product: `(y2-y1)*x3 + (x1-x2)*y3 + (x2*y1 - x1*y2)`. Uses `FixedMulShift(..., 16)` for fixed-point precision.

## Control Flow Notes
This is a support library, not part of the main game loop. Functions are called on-demand:
- **Startup:** `StartupSoftError()`, `FindEGAColors()`, `UL_ChangeDirectory()` for initialization
- **Runtime:** File I/O (LoadFile/SaveFile), math utilities (FindDistance, atan2_appx), memory management (SafeMalloc)
- **Error paths:** `Error()` performs emergency shutdown, logging, and display before exit
- **Shutdown:** `ShutdownSoftError()` closes debug/error files

## External Dependencies
- **System headers:** `stdio.h`, `stdlib.h`, `string.h`, `malloc.h`, `dos.h`, `fcntl.h`, `errno.h`, `io.h`, `ctype.h`, `direct.h`, `sys/stat.h`
- **Game-specific:** `z_zone.h` (Z_Malloc, Z_Free, Z_LevelMalloc, zonememorystarted), `rt_in.h` (IN_UpdateKeyboard, Keyboard array), `rt_vid.h` (VL_ClearVideo, possibly gamma tables), `rt_main.h` (ShutDown, gamestate, player), `rt_dr_a.h`, `rt_playr.h`, `scriplib.h` (GetToken, script variables), `rt_menu.h`, `rt_cfg.h`, `rt_view.h`, `modexlib.h`, `version.h` (ROTT_ERR, version constants)
- **Engine headers:** `watcom.h` (FixedMul, FixedDiv2, FixedMulShift), `develop.h` (development/debug macros)
- **Externals:** `_argc`, `_argv` (command-line), `player` (player object), `gamestate` (game state record), `gammatable`, `gammaindex`, `SOUNDSETUP`
