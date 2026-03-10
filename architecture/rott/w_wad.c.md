# rott/w_wad.c

## File Purpose
WAD (Where's All the Data) file manager for loading and accessing game lumps. Implements virtual filesystem for multi-lump WAD files and single-file lumps, with demand-loaded caching integrated into the zone memory system. Includes optional data corruption detection via CRC checksums.

## Core Responsibilities
- Load WAD files (multi-lump archives) and individual lumps, parsing headers and directory tables
- Maintain global lump registry with file handle, position, and size metadata
- Provide name-to-index and index-to-name lookup services for lumps
- Implement demand-based caching with zone memory tag integration
- Detect modified WADs via CRC checksum verification
- Support lump read/write operations with bounds checking and I/O error detection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `lumpinfo_t` | struct | Metadata for a single lump: file handle, disk position, size, 8-char name |
| `wadinfo_t` | struct | WAD file header: "IWAD" identification, lump count, info table offset |
| `filelump_t` | struct | Directory entry in WAD file: file position, size, 8-char name |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `numlumps` | int | global | Total number of loaded lumps across all files |
| `lumpcache` | void ** | global | Array of cached lump data pointers, indexed by lump number |
| `lumpinfo` | lumpinfo_t * | static | Dynamically allocated array of lump metadata; reallocated as files added |
| `lumpcheck` | byte * | static | Data corruption test array; only allocated if `DATACORRUPTIONTEST == 1` |
| `readinglump` | int | static | Current lump being read (for error reporting) |
| `lumpdest` | byte * | static | Destination buffer for read operation (for error reporting) |

## Key Functions / Methods

### W_AddFile
- **Signature:** `void W_AddFile(char *filename)`
- **Purpose:** Load a single WAD file or standalone lump into the system, extending the global lump registry.
- **Inputs:** Filename string (may have `.wad`, `.rts`, or other extension).
- **Outputs/Return:** None; updates `numlumps` and reallocates `lumpinfo`.
- **Side effects:** Opens file handle (kept open for lump reads); calls `Z_Realloc()` on `lumpinfo`; prints load status if not `quiet`.
- **Calls:** `open()`, `filelength()`, `read()`, `lseek()`, `ExtractFileBase()`, `Z_Realloc()`, `strncpy()`, `strcmpi()`, `strncmp()`.
- **Notes:** WAD files must have "IWAD" header; single-file lumps use the basename as lump name. Lump names can override earlier ones (last-loaded wins).

### W_InitMultipleFiles
- **Signature:** `void W_InitMultipleFiles(char **filenames)`
- **Purpose:** Initialize the lump system with multiple files (null-terminated array). Calls `W_AddFile()` for each.
- **Inputs:** Null-terminated array of filename strings.
- **Outputs/Return:** None; initializes `numlumps`, `lumpinfo`, `lumpcache`, and optional `lumpcheck`.
- **Side effects:** Allocates global `lumpcache` array; calls `W_AddFile()` for each file; calls `W_CheckWADIntegrity()` unless `SOUNDSETUP` is set.
- **Calls:** `SafeMalloc()`, `W_AddFile()`, `calloc()`, `W_CheckWADIntegrity()`.
- **Notes:** At least one file must be found or function errors.

### W_InitFile
- **Signature:** `void W_InitFile(char *filename)`
- **Purpose:** Convenience wrapper to initialize from a single file.
- **Inputs:** Filename string.
- **Outputs/Return:** None.
- **Calls:** `W_InitMultipleFiles()`.

### W_CheckNumForName
- **Signature:** `int W_CheckNumForName(char *name)`
- **Purpose:** Look up a lump by 8-character name (case-insensitive). Returns lump index or -1 if not found.
- **Inputs:** Lump name (any length; truncated to 8 chars).
- **Outputs/Return:** Lump index (0–`numlumps`-1) or -1 if not found.
- **Side effects:** None.
- **Calls:** `strncpy()`, `strupr()`.
- **Notes:** Optimizes comparison by casting 8-byte name to two 32-bit integers. Scans linearly; later lumps override earlier ones if duplicate names exist.

### W_GetNumForName
- **Signature:** `int W_GetNumForName(char *name)`
- **Purpose:** Get lump index, fatal error if not found.
- **Inputs:** Lump name.
- **Outputs/Return:** Lump index.
- **Calls:** `W_CheckNumForName()`, `Error()`.

### W_ReadLump
- **Signature:** `void W_ReadLump(int lump, void *dest)`
- **Purpose:** Read lump data from disk into a buffer.
- **Inputs:** Lump index, destination buffer (must be ≥ `W_LumpLength(lump)` bytes).
- **Outputs/Return:** None; data written to `dest`.
- **Side effects:** Updates `readinglump` and `lumpdest` globals (for error reporting); seeks and reads file handle.
- **Calls:** `lseek()`, `read()`, `Error()`.
- **Notes:** Bounds-checks lump index; fatals if read count doesn't match lump size.

### W_WriteLump
- **Signature:** `void W_WriteLump(int lump, void *src)`
- **Purpose:** Write lump data from a buffer to disk.
- **Inputs:** Lump index, source buffer.
- **Outputs/Return:** None.
- **Side effects:** Seeks and writes file handle.
- **Calls:** `lseek()`, `write()`, `Error()`.
- **Notes:** Bounds-checks lump index; fatals if write count mismatch.

### W_CacheLumpNum
- **Signature:** `void *W_CacheLumpNum(int lump, int tag)`
- **Purpose:** Load and cache a lump with zone memory tagging; return pointer to cached data.
- **Inputs:** Lump index, memory tag (for zone allocator).
- **Outputs/Return:** Pointer to cached lump data in zone heap.
- **Side effects:** Allocates zone memory; calls `Z_Malloc()`, `Z_ChangeTag()`, `W_ReadLump()`; if `DATACORRUPTIONTEST`, computes and stores CRC on load and verifies on reaccess.
- **Calls:** `W_LumpLength()`, `Z_Malloc()`, `W_ReadLump()`, `CalculateCRC()`, `Z_ChangeTag()`, `W_GetNameForNum()`, `SoftError()`, `Error()`.
- **Notes:** If lump already cached, updates its zone tag; if `DATACORRUPTIONTEST`, periodically re-checks CRC to detect memory corruption.

### W_CacheLumpName
- **Signature:** `void *W_CacheLumpName(char *name, int tag)`
- **Purpose:** Cache by name; convenience wrapper.
- **Inputs:** Lump name, memory tag.
- **Outputs/Return:** Pointer to cached data.
- **Calls:** `W_GetNumForName()`, `W_CacheLumpNum()`.

---

## Control Flow Notes
**Initialization (Game Startup):**  
`W_InitMultipleFiles()` → loops `W_AddFile()` → builds `lumpinfo` registry → initializes `lumpcache` → optionally checks WAD integrity.

**Runtime (Frame/Update):**  
Code calls `W_CacheLumpNum()` or `W_CacheLumpName()` on-demand. If not already cached, zone allocator loads the lump into memory and stores handle in `lumpcache[lump_index]`. Subsequent accesses reuse cached data or update its zone tag.

**Shutdown:**  
Zone memory manager frees tagged lumps; file handles left open (implicit close on exit).

## External Dependencies
- **Standard C:** `<stdio.h>`, `<conio.h>`, `<string.h>`, `<malloc.h>`, `<io.h>`, `<fcntl.h>`, `<sys/stat.h>`
- **Local headers:** `rt_def.h` (constants, types), `rt_util.h` (utility functions), `_w_wad.h` (private types: `lumpinfo_t`, `wadinfo_t`, `filelump_t`), `z_zone.h` (memory manager), `rt_crc.h` (CRC calculation), `rt_main.h`, `isr.h`, `develop.h` (configuration macros).
- **External functions used:** `SafeMalloc()`, `ExtractFileBase()`, `Z_Realloc()`, `Z_Malloc()`, `Z_ChangeTag()`, `CalculateCRC()`, `Error()`, `SoftError()` (defined elsewhere).
- **POSIX I/O:** `open()`, `close()`, `read()`, `write()`, `lseek()`, `fstat()` (conditional NeXT compatibility shims provided).
