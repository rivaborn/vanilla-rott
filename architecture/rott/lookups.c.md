# rott/lookups.c

## File Purpose
Standalone utility that generates lookup tables for the ROTT renderer (pixel angles, sine, tangent, and gamma correction tables). These tables are precomputed at build time and written to a binary file for use by the game engine during runtime.

## Core Responsibilities
- Calculate pixel-to-angle mapping for raycasting (perspective correction)
- Generate sine and tangent lookup tables for trigonometric calculations
- Produce gamma correction curves for display brightness adjustment
- Write all four lookup tables to a binary output file in a fixed format

## Key Types / Data Structures
None (all data stored in module-level arrays).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `pangle` | `fixed[512]` | file-static | Pixel angle lookup table; maps screen columns to fine angles |
| `sintable` | `long[2561]` | file-static | Sine table covering 0°–90° plus full circle mirrors |
| `tantable` | `short[2048]` | file-static | Tangent table for 0°–360° |
| `gammatable` | `byte[512]` | file-static | Gamma correction table (8 levels × 64 entries each) |
| `_argc`, `_argv` | extern | external | Command-line arguments (filename for output) |

## Key Functions / Methods

### CalcPixelAngles
- **Signature:** `void CalcPixelAngles(void)`
- **Purpose:** Calculate screen-pixel-to-angle mapping for raycaster's per-pixel perspective correction.
- **Inputs:** None (uses hardcoded constants FINEANGLES, PANGLES, FPFOCALWIDTH, PI).
- **Outputs/Return:** None; populates `pangle[]` array.
- **Side effects:** Writes to global `pangle[]` array.
- **Calls:** `atan()` (math library).
- **Notes:** 
  - Offsets each pixel by +0.5 to center the sample within the pixel.
  - Uses `atan()` to compute angle from horizontal tangent.
  - Converts angle to fixed-point integer using radtoint scaling factor.

### BuildSinTable
- **Signature:** `void BuildSinTable(void)`
- **Purpose:** Generate sine lookup table by sampling sin(angle) at regular intervals.
- **Inputs:** None (uses PI, FINEANGLES, FINEANGLEQUAD, GLOBAL1 constants).
- **Outputs/Return:** None; populates `sintable[]` array.
- **Side effects:** Writes to global `sintable[]` array.
- **Calls:** `sin()` (math library).
- **Notes:**
  - Stores sin for 0°–90° in quadrant order, then mirrors across axes for full 360°.
  - Uses fixed-point scaling (GLOBAL1 = 1<<16) for integer math.
  - Symmetric: negative values stored for 180°–360°.

### BuildTanTable
- **Signature:** `void BuildTanTable(void)`
- **Purpose:** Generate tangent lookup table for 0°–360°.
- **Inputs:** None (uses PI, FINEANGLES, GLOBAL1 constants).
- **Outputs/Return:** None; populates `tantable[]` array.
- **Side effects:** Writes to global `tantable[]` array.
- **Calls:** `tan()` (math library).
- **Notes:**
  - Stores as `short` (half-precision fixed-point: `>>1` shift).
  - Samples at regular intervals across full 360°.

### BuildGammaTable
- **Signature:** `void BuildGammaTable(void)`
- **Purpose:** Generate gamma correction curves for display brightness adjustment.
- **Inputs:** None (uses NUMGAMMALEVELS=8 constant).
- **Outputs/Return:** None; populates `gammatable[]` array.
- **Side effects:** Writes to global `gammatable[]` array; increments file-local counter `j`.
- **Calls:** `pow()` (math library).
- **Notes:**
  - Generates 8 separate gamma curves (gGamma starts at 0x100, increments by 32).
  - Each curve has 64 entries (brightness levels 0–63).
  - Clamps output to [0, 63] range.

### Error
- **Signature:** `void Error(char *error, ...)`
- **Purpose:** Print formatted error message and exit program.
- **Inputs:** Format string and variable arguments (variadic).
- **Outputs/Return:** None; exits with code 1.
- **Side effects:** Prints to stdout; calls `exit(1)`.
- **Calls:** `va_start()`, `vprintf()`, `va_end()`, `exit()`.
- **Notes:** Appends newline after message.

### SafeOpenWrite
- **Signature:** `int SafeOpenWrite(char *filename)`
- **Purpose:** Open file for writing with error checking.
- **Inputs:** Filename string.
- **Outputs/Return:** File descriptor (handle).
- **Side effects:** Creates/truncates file; calls `Error()` on failure.
- **Calls:** `open()` (POSIX), `strerror()`.
- **Notes:** Uses `O_RDWR | O_BINARY | O_CREAT | O_TRUNC` flags; exits on failure.

### SafeWrite
- **Signature:** `void SafeWrite(int handle, void *buffer, long count)`
- **Purpose:** Write data to file in chunks (handles large writes via multiple system calls).
- **Inputs:** File descriptor, buffer pointer, byte count.
- **Outputs/Return:** None.
- **Side effects:** Writes to file; calls `Error()` on write failure.
- **Calls:** `write()` (POSIX).
- **Notes:** Chunks writes into 0x8000-byte (32 KB) chunks; advances buffer pointer between calls.

### main
- **Signature:** `void main(void)`
- **Purpose:** Entry point; orchestrates generation and serialization of all lookup tables.
- **Inputs:** Command-line arguments (`_argc`, `_argv`).
- **Outputs/Return:** None; exits with status code.
- **Side effects:** Calls all build functions; creates binary output file; exits on success or error.
- **Calls:** `SafeOpenWrite()`, `CalcPixelAngles()`, `BuildSinTable()`, `BuildTanTable()`, `BuildGammaTable()`, `SafeWrite()` (4 table writes + 4 size writes), `close()`.
- **Notes:**
  - Expects exactly 2 command-line arguments (`<progname> <outputfile>`).
  - Writes file format: for each table, writes `(int)size` followed by table data.
  - Exits with status 0 on success.

## Control Flow Notes
This is a **build-time utility** (not part of the runtime game loop). Execution is linear and one-shot:
1. Validate command-line arguments.
2. Open output file.
3. Compute all four lookup tables in order.
4. Serialize each table with its size prefix.
5. Close file and exit.

Tables are later loaded by the renderer at runtime (via game engine initialization).

## External Dependencies
- **Includes:**
  - `rt_def.h`, `rt_util.h`, `rt_view.h` – local engine headers (defines constants like PANGLES, FINEANGLES, GAMMAENTRIES, GLOBAL1, PI).
  - `<math.h>` – Standard C math library (`sin()`, `tan()`, `atan()`, `pow()`).
  - `<dos.h>`, `<conio.h>`, `<io.h>` – DOS/legacy I/O APIs (`open()`, `write()`, `close()`).
  - `<fcntl.h>`, `<errno.h>`, `<sys/stat.h>` – POSIX file constants and error codes.
  - `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<stdarg.h>`, `<ctype.h>` – Standard C library.
  - `memcheck.h` – Memory debugging tool (no-op macros when disabled).
- **Defined elsewhere:**
  - `Error()`, `SafeOpenWrite()`, `SafeRead()`, `SafeWrite()` – Declared in `rt_util.h` header; implemented locally.
