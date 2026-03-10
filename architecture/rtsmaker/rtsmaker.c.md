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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `lumpinfo_t` | struct | Metadata for a single lump: file offset, size, 8-char name |
| `wadinfo_t` | struct | WAD file header: identification string, lump count, offset to lump directory |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `destpath` | char[MAXPATH] | static | Output RTS file path |
| `newwad` | wadinfo_t | static | WAD header being written to output |
| `newlumps` | lumpinfo_t\* | static | Array of lump metadata for output file |
| `newhandle` | int | static | Open file handle for output RTS file |
| `lump_p` | lumpinfo_t\* | static | Pointer to current lump being processed |
| `lumpon` | int | static | Current lump index/count |
| `lumppath` | char[MAXPATH] | static | Path to current input sound file |
| `datahandle` | int | static | File handle for input (sound file) |
| `scriptfilename` | char[MAXPATH] | static | Path to script file |
| `sourcepath` | char[MAXPATH] | static | Source directory for data files |
| `defaultdestpath` | char[MAXPATH] | static | Destination directory for output RTS |
| `lumpscopied` | int | static | Counter of lumps successfully copied |
| `loadwad` | wadinfo_t | static | WAD header of input file (unused, #if 0) |
| `wadlumps` | lumpinfo_t\* | static | Lump array from input WAD (unused, #if 0) |
| `inwadfile` | boolean | static | Flag: input is a WAD file (unused, #if 0) |

## Key Functions / Methods

### main
- **Signature:** `int main(int argc, char **argv)`
- **Purpose:** Entry point; parses command-line arguments and orchestrates either unpacking or script-based RTS creation
- **Inputs:** Command-line arguments (either `-u filename` for unpack mode or script file for creation mode)
- **Outputs/Return:** Exit code (0 on success, 1 on error)
- **Side effects:** Reads and writes files; prints usage and status messages; calls `ProcessScript()` or `UnpackRTS()`
- **Calls:** `CheckParm()`, `UnpackRTS()`, `getcwd()`, `LoadScriptFile()`, `ProcessScript()`, `WriteDirectory()`
- **Notes:** Help text shown if `-?` flag or no args; Windows-centric path handling with backslashes

### ProcessScript
- **Signature:** `void ProcessScript(void)`
- **Purpose:** Main script processing loop; reads script tokens, initializes RTS output, and copies all sound lumps
- **Inputs:** Global `scriptfilename` loaded via `LoadScriptFile()`; token stream via `GetToken()`
- **Outputs/Return:** None (modifies global output file state)
- **Side effects:** Creates and writes to output RTS file; opens multiple input sound files; updates global `lump_p`, `lumpon`, `lumpscopied`
- **Calls:** `GetToken()`, `CmdRTSName()`, `AddLabel()`, `DefaultPath()`, `DefaultExtension()`, `ExtractFileBase()`, `OpenLump()`, `CopyLump()`, `unlink()`, `Error()`
- **Notes:** Expects exactly `MAXSOUNDS` (10) sound file references; fails with error if fewer provided; adds "REMOSTRT" and "REMOSTOP" boundary labels

### CmdRTSName
- **Signature:** `void CmdRTSName(char *name)`
- **Purpose:** Initialize the output RTS file; allocate lump table and seek past header
- **Inputs:** `name` — output RTS filename (without extension)
- **Outputs/Return:** None (modifies globals: `destpath`, `newhandle`, `newwad`, `newlumps`, `lump_p`, `lumpon`)
- **Side effects:** Creates/opens output file; allocates 0x2000 bytes for lump directory; seeks file pointer to start of data
- **Calls:** `DefaultPath()`, `DefaultExtension()`, `SafeOpenWrite()`, `SafeMalloc()`, `lseek()`, `printf()`
- **Notes:** Hardcodes "IWAD" identification; allocates fixed 0x2000 buffer for lump table

### OpenLump
- **Signature:** `void OpenLump(void)`
- **Purpose:** Open an input sound file for reading (WAD support #if 0 disabled)
- **Inputs:** Global `lumppath` — path to sound file; global `inwadfile` flag (unused)
- **Outputs/Return:** None (modifies global `datahandle`)
- **Side effects:** Opens file for reading; seeks to file start if in WAD (disabled)
- **Calls:** `SafeOpenRead()`, `lseek()` (if WAD support enabled)
- **Notes:** Simplified implementation; WAD file support is #if 0 disabled; only handles separate files

### CopyLump
- **Signature:** `void CopyLump(void)`
- **Purpose:** Copy lump data from input file to output RTS file in chunks
- **Inputs:** Global `datahandle`, `lump_p`, `lumpon`, `newhandle`
- **Outputs/Return:** None (writes to output file; updates `lump_p->filepos`, `lump_p->size`)
- **Side effects:** Reads from input; writes to output; allocates temporary buffer; increments `lumpon`, advances `lump_p`; prints status
- **Calls:** `filelength()`, `tell()`, `LittleLong()`, `SafeMalloc()`, `SafeRead()`, `SafeWrite()`, `SafeFree()`, `printf()`
- **Notes:** Copies in 16KB chunks (MAXCOPYSIZE); stores file offset and size in little-endian format

### UnpackRTS
- **Signature:** `void UnpackRTS(char *filename)`
- **Purpose:** Extract all lumps from an RTS file and write them as individual files with auto-detected extension (.voc or .wav)
- **Inputs:** `filename` — path to RTS file to unpack
- **Outputs/Return:** None (writes extracted files to disk)
- **Side effects:** Opens RTS file; allocates lump table; creates individual files for each lump; reads first byte to detect format
- **Calls:** `SafeOpenRead()`, `SafeRead()`, `LittleLong()`, `lseek()`, `Error()`, `SafeMalloc()`, `SafeOpenWrite()`, `SafeWrite()`, `SafeFree()`, `close()`, `printf()`
- **Notes:** Inspects first byte of lump: 'C' = .voc, otherwise = .wav; skips empty lumps; closes files explicitly

### AddLabel
- **Signature:** `void AddLabel(char *string)`
- **Purpose:** Add a label lump entry (zero-size marker) to the output
- **Inputs:** `string` — label name (up to 8 chars)
- **Outputs/Return:** None (advances `lump_p`, increments `lumpon`)
- **Side effects:** Writes zero filepos/size; copies name; updates lump pointer and count
- **Calls:** `strupr()`, `strncpy()`
- **Notes:** Creates a label-only lump with no associated data; used for REMOSTRT/REMOSTOP boundaries

### WriteDirectory
- **Signature:** `void WriteDirectory(void)`
- **Purpose:** Finalize RTS file by writing the lump directory table and WAD header
- **Inputs:** Global state: `newwad`, `newlumps`, `lumpon`, `newhandle`
- **Outputs/Return:** None (writes to output file and closes it)
- **Side effects:** Seeks to end of file; writes lump table; seeks to file start; writes WAD header; closes output file
- **Calls:** `LittleLong()`, `tell()`, `write()`, `lseek()`
- **Notes:** Converts counts/offsets to little-endian before writing

## Control Flow Notes
**Initialization → Script Processing → Finalization**

1. **main()** parses arguments; checks for unpack mode (`-u`) or standard mode
2. In standard mode: loads script file, prints paths, calls `ProcessScript()`
3. **ProcessScript()** reads RTS output filename, creates file via `CmdRTSName()`, then iterates reading sound file paths and copying each
4. Each sound file is opened (`OpenLump()`) and copied (`CopyLump()`)
5. **WriteDirectory()** finalizes by writing lump table and header
6. In unpack mode: **UnpackRTS()** reads an existing RTS and extracts all lumps to files

## External Dependencies
- **cmdlib.h**: `SafeOpenRead()`, `SafeOpenWrite()`, `SafeRead()`, `SafeWrite()`, `SafeMalloc()`, `SafeFree()`, `Error()`, `CheckParm()`, `DefaultPath()`, `DefaultExtension()`, `ExtractFileBase()`, `LittleLong()`
- **scriplib.h**: `LoadScriptFile()`, `GetToken()`, global `token[]`, global `endofscript` flag
- **Standard C**: `stdlib.h`, `stdio.h`, `string.h` (file I/O, memory, string utilities)
- **Win32-specific**: `<io.h>`, `<fcntl.h>`, `<direct.h>` (file handles, getcwd, close, unlink)
